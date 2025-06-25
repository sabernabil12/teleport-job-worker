# Job Worker Service: Design Document

## Table of Contents 

- [Job Worker Service: Design Document](#job-worker-service-design-document)
  - [Table of Contents 1](#table-of-contents-1)
  - [Overview](#overview)
  - [Scope](#scope)
  - [CLI User Experience](#cli-user-experience)
    - [Command Syntax](#command-syntax)
    - [Key Features](#key-features)
      - [**Security \& Authentication**](#security--authentication)
      - [**Performance \& Reliability**](#performance--reliability)
      - [**User Experience**](#user-experience)
      - [**Architecture \& Design**](#architecture--design)
    - [Example Commands](#example-commands)
  - [Proposed API](#proposed-api)
    - [gRPC Service Definition](#grpc-service-definition)
    - [Job Status Enum](#job-status-enum)
      - [Messages](#messages)
  - [Design Approach](#design-approach)
  - [Graceful Shutdown](#graceful-shutdown)
    - [Shutdown Sequence](#shutdown-sequence)
    - [Job Cleanup](#job-cleanup)
    - [Timeout Management](#timeout-management)
    - [Implementation Details](#implementation-details)
    - [**Stream Termination**:](#stream-termination)
  - [Implementation Details](#implementation-details-1)
    - [Worker Library](#worker-library)
    - [API Server](#api-server)
    - [CLI](#cli)
  - [Output Streaming Architecture](#output-streaming-architecture)
    - [**Implementation Details**](#implementation-details-2)
    - [**Data Flow**](#data-flow)
    - [**Key Benefits**](#key-benefits)
    - [**Why Not Other Approaches**](#why-not-other-approaches)
    - [**Why this approach is efficient**](#why-this-approach-is-efficient)
  - [Job Stopping \& Termination](#job-stopping--termination)
    - [**Job Stop Methods**](#job-stop-methods)
    - [**StopJob Implementation**](#stopjob-implementation)
    - [**Process Cleanup**](#process-cleanup)
    - [**Error Handling**](#error-handling)
    - [**Implementation Details**](#implementation-details-3)
  - [gRPC Streaming Lifecycle Management](#grpc-streaming-lifecycle-management)
    - [**Stream Lifecycle**](#stream-lifecycle)
    - [**Client Disconnection Handling**](#client-disconnection-handling)
    - [**Stream Patterns**](#stream-patterns)
    - [**Graceful Shutdown Handling**](#graceful-shutdown-handling)
  - [Security Considerations](#security-considerations)
    - [Security Implementation Details](#security-implementation-details)
  - [Edge Cases \& Error Handling](#edge-cases--error-handling)
  - [Implementation Plan](#implementation-plan)
    - [**PR Breakdown**](#pr-breakdown)
    - [**Testing Approach**](#testing-approach)
    - [**Output Guarantee from Start**](#output-guarantee-from-start)
  - [Conclusion](#conclusion)

---

## Overview

This document proposes a design for a prototype job worker service that provides an API to run arbitrary Linux processes. The service is composed of three main components:

1. **Worker Library**: Core logic for job management and process execution.
2. **API Server**: gRPC server exposing job management APIs, secured with mTLS.
3. **CLI Client**: Command-line tool to interact with the API server.

The system is designed for security, efficiency, and extensibility, with a focus on robust process management, secure communication, and an easy-to-use and clear user experience.

---

## Scope

- **In Scope**:
  - Running, stopping, and querying the status of arbitrary Linux processes.
  - Streaming process output (stdout/stderr) from the start of execution.
  - Secure gRPC API with mTLS authentication and strong TLS configuration.
  - Simple, robust authorization scheme.
  - CLI for job management and output streaming.
  - Efficient, concurrent handling of multiple jobs and clients.
  - Tests for key components (authorization, output streaming).

- **Out of Scope**:
  - Containerization or sandboxing of jobs.
  - Advanced scheduling or job dependencies.
  - Persistent storage of job history beyond process lifetime.
  - Limitation on number of concurrent jobs.
  - Environment variable support.

---

## CLI User Experience

### Command Syntax

The CLI uses a simplified syntax:

- **Start Job**: `./build/cli start <command> [args...]`
- **Get Status**: `./build/cli status <job-id>`
- **Stream Output**: `./build/cli stream <job-id>`
- **Stop Job**: `./build/cli stop <job-id>`

### Key Features

#### **Security & Authentication**
- **mTLS Communication**: TLS 1.3 with client certificate verification.
- **Hardcoded Authentication**: Client certificate subjects validated against hardcoded allowlist.
- **Command-based Authorization**: Users are restricted to specific allowed commands based on their identity.
- **Input Validation**: All user input (commands, arguments) is validated and sanitized. EG: check for forbidden characters, length check etc.
- **Process Isolation**: Jobs run as isolated child processes without shell interpretation.

#### **Performance & Reliability**
- **Real-time Output Streaming**: Live job output via gRPC streaming with proper lifecycle management.
- **Concurrent Job Support**: Multiple jobs and clients handled simultaneously.
- **UUID-based Job IDs**: Prevents race conditions and ensures uniqueness.
- **Graceful Shutdown**: Signal handling with job cleanup and 30-second timeout.
- **Unbounded Output Buffering**: Ensures full output capture without data loss.

#### **User Experience**
- **Simple CLI Interface**: Intuitive commands without complex flags.
- **Real-time Output**: Job output streamed to terminal immediately.
- **Comprehensive Status**: Job status, exit codes, start/end times.
- **Cross-platform**: Works on Linux, macOS, and other Unix-like systems.

#### **Architecture & Design**
- **Direct Pipe with Multiplexing**: Efficient output streaming architecture.
- **Automatic Cleanup**: Proper process reaping and resource management.
- **Extensible Design**: Clean separation between worker, server, and client components.

### Example Commands

```sh
# Start a job
./build/cli start echo "Hello World"
./build/cli start ls -la

# Get job status
./build/cli status 550e8400-e29b-41d4-a716-446655440000

# Stream job output
./build/cli stream 550e8400-e29b-41d4-a716-446655440000

# Stop a job
./build/cli stop 550e8400-e29b-41d4-a716-446655440000
```

---

## Proposed API

### gRPC Service Definition

```protobuf
service JobWorker {
  rpc StartJob(StartJobRequest) returns (StartJobResponse);
  rpc StopJob(StopJobRequest) returns (StopJobResponse);
  rpc GetJobStatus(GetJobStatusRequest) returns (GetJobStatusResponse);
  rpc StreamJobOutput(StreamJobOutputRequest) returns (stream StreamJobOutputResponse);
}
```

### Job Status Enum

```protobuf
enum JobStatus {
  JOB_STATUS_UNSPECIFIED = 0;  // Default value, should not be used
  JOB_STATUS_CREATED = 1;      // Job created but not started
  JOB_STATUS_RUNNING = 2;      // Job is currently executing
  JOB_STATUS_COMPLETED = 3;    // Job finished successfully
  JOB_STATUS_ERROR = 4;        // Job failed or encountered an error
  JOB_STATUS_STOPPED = 5;      // Job was manually stopped
}
```

#### Messages

- `StartJobRequest`: Command (string), arguments ([]string).
- `StartJobResponse`: Job ID (string), status (JobStatus), start time (string).
- `StopJobRequest`: Job ID (string).
- `StopJobResponse`: Success/failure (bool), end time (string).
- `GetJobStatusRequest`: Job ID (string).
- `GetJobStatusResponse`: Status (JobStatus), exit code (int32), start time (string), end time (string).
- `StreamJobOutputRequest`: Job ID (string).
- `StreamJobOutputResponse`: Data (bytes), timestamp (string).

---

## Design Approach

- **Process Management**: Each job is a Linux process, managed by the worker library. Jobs are tracked by UUIDs (e.g., `550e8400-e29b-41d4-a716-446655440000`) to avoid race conditions in concurrent environments. Output is captured and made available for streaming.
- **Concurrency**: The system supports multiple concurrent jobs and clients, using Go's concurrency primitives.
- **Output Streaming**: Output is streamed (as soon as it is produced) efficiently using pipes and channels, avoiding polling or busy-waiting. 
- **Security**: All API communication is secured with mTLS. Only authorized clients can manage jobs.

---

## Graceful Shutdown

The server implements graceful shutdown to ensure data integrity and proper resource cleanup:

### Shutdown Sequence

1. **Stop accepting new connections** - gRPC server stops accepting new requests.
2. **Wait for active requests** - Allows ongoing gRPC calls to complete (30-second timeout).
3. **Job cleanup** - Stops all running jobs gracefully.
4. **Resource cleanup** - Closes connections and releases resources.

### Job Cleanup

- Iterates through all active jobs.
- Stops running jobs using process termination.
- Logs cleanup progress and any errors.
- Ensures no orphaned processes remain.

### Timeout Management

- 30-second timeout for graceful shutdown.
- Falls back to forced shutdown if timeout exceeded.
- Prevents indefinite hanging during shutdown.

### Implementation Details

- Uses Go's `signal.Notify()` for signal handling.
- Leverages gRPC's `GracefulStop()` for connection management.
- Thread-safe job cleanup with mutex protection.
- Comprehensive logging for debugging and monitoring.

### **Stream Termination**: 

Streams end when job completes, client disconnects, server shuts down, or error occurs. Client channels are automatically removed from broadcast list and closed.

---

## Implementation Details

### Worker Library

- **Job Management**: Maintains a map of job UUIDs to process handles and metadata. Uses atomic UUID generation to avoid race conditions in concurrent job creation.
- **Process Management**: Each job creates a child process using `exec.Command()`.
- **Process Reaping**: For every job, a dedicated goroutine calls `Cmd.Wait()` as soon as the process exits, ensuring all child processes are properly reaped and no zombies remain.
- **Output Streaming**: Implements Direct Pipe with Broadcast architecture (see [Output Streaming Architecture](#output-streaming-architecture) for detailed technical implementation). Uses `os.Pipe` to capture stdout and stderr from child processes. Both stdout and stderr are combined into a single broadcast stream that sends raw bytes to all connected clients without assumptions about content type (text/binary). Each client gets their own dedicated channel, ensuring no data stealing between clients. Output is captured using two goroutines that read from process pipes (stdout and stderr) and broadcast to all client channels, enabling real-time streaming without polling or busy-waiting. The capture starts immediately when the process begins, ensuring no output is lost.
- **Concurrency**: Uses goroutines and channels for process management and output streaming.

### API Server

- **gRPC**: Exposes job management APIs.
- **TLS**: Configured with strong cipher suites, client cert verification, and secure key storage.
- **Hardcoded Authentication**: Client certificate subjects are validated against hardcoded allowlist before allowing access to any API.
- **Command-based Authorization**: Users are restricted to specific allowed commands based on their identity. Authorization logic is separated into `auth.go` for better code organization. Multiple user profiles are supported:
  - **Admin Profile** (`CN=admin`): Full system access including `ps`, `top`, `df`, `du`, `who`, `w`.
  - **Developer Profile** (`CN=developer`): Development tools access including `git`, `go`, `make`.
  - **Read-only Profile** (`CN=readonly`): Limited to basic read operations like `echo`, `ls`, `cat`, `head`, `tail`.
  - **Default Profile**: Safe commands for users not explicitly configured.

### CLI

- **User Experience**: Simple, consistent commands for job management.
- **mTLS**: Requires client cert/key and CA cert for all operations.
- **Output Streaming**: Streams output to stdout/stderr in real time.

---

## Output Streaming Architecture

The system implements **Direct Pipe with Broadcast** for efficient real-time output streaming. Here's the concrete architecture:

### **Implementation Details**

1. **Process Output Capture**:
   - Each job creates child process using `exec.Command()`.
   - `StdoutPipe()` and `StderrPipe()` create OS-level pipes connected to the child process.
   - Two separate goroutines run `io.Copy(&broadcastWriter{job}, pipe)` for stdout and stderr.
   - Both stdout and stderr are combined into a single broadcast stream.
   - Raw bytes are immediately broadcast to all connected clients.

2. **Broadcast-Based Multiplexing**:
   - Each job maintains a map of client channels (`map[chan []byte]bool`).
   - `broadcastWriter` implements `io.Writer` interface to broadcast to all clients.
   - Each client gets their own dedicated unbuffered channel.
   - Data is copied for each client to prevent interference between clients.
   - Slow clients are skipped (non-blocking send) to avoid blocking the broadcast.

3. **gRPC Streaming Layer**:
   - Server maintains a map of job UUIDs to worker.Job instances.
   - `StreamJobOutput` RPC creates a dedicated client channel and adds it to the job's broadcast list.
   - Each client gets an independent gRPC stream that reads from their dedicated channel.
   - Client channels are automatically cleaned up when clients disconnect or job completes.

### **Data Flow**
```
Child Process (stdout + stderr) 
    ↓ (OS pipes)
io.Copy() goroutines (2x - stdout + stderr)
    ↓ (broadcastWriter)
Broadcast to All Client Channels
    ↓ (one channel per client)
gRPC Stream Clients (independent)
```

### **Key Benefits**
- **Real-time**: Output appears immediately as process produces it.
- **Concurrent**: Multiple clients can stream the same job output without data stealing.
- **Full output capture**: Each client receives all output from the start.
- **Automatic cleanup**: Client channels are closed when clients disconnect or job completes.
- **Simplified**: Single output stream eliminates complexity of separate stdout/stderr handling.
- **No interference**: Each client gets their own data copy, preventing data stealing between clients.

### **Why Not Other Approaches**
- **File + fsnotify**: Would require disk I/O and file management overhead.
- **Ring buffer**: More complex, doesn't provide the same real-time guarantees.
- **Separate stdout/stderr channels**: Added complexity without significant benefit for most use cases.
- **Shared single channel**: Multiple clients reading from same channel would steal data from each other.

### **Why this approach is efficient**
- **Uses blocking I/O:** Goroutines read from OS pipes and sleep until new output is available, consuming no CPU while idle.
- **No polling or busy-waiting:** The system reacts instantly to new data, rather than repeatedly checking for it.
- **Real-time delivery:** Output is pushed to clients as soon as it is produced, ensuring low latency.
- **Resource efficient:** CPU and memory usage remain minimal, as work only occurs when there is actual process output.

---

## Job Stopping & Termination

The system provides comprehensive job stopping capabilities with proper process management and cleanup:

### **Job Stop Methods**

1. **Client-Initiated Stop**: Clients can explicitly stop running jobs using the `StopJob` RPC.
2. **Server Shutdown**: All running jobs are stopped during graceful shutdown.
3. **Process Completion**: Jobs naturally terminate when their process completes.

### **StopJob Implementation**

**Process Termination**:
- Uses `Process.Kill()` to send SIGKILL to the job's process.
- Ensures immediate termination of the child process.
- Updates job status to `JOB_STATUS_STOPPED` and records end time.
- Closes the output channel to signal end-of-stream to all clients.

**Response**:
- Returns success/failure status.
- Includes end time when the job was stopped.
- Provides clear error messages if job doesn't exist or isn't running.

### **Process Cleanup**

**Automatic Reaping**:
- Each job has a dedicated goroutine running `Cmd.Wait()` to reap the process.
- Ensures no zombie processes remain in the system.
- Handles both natural completion and forced termination.

**Resource Cleanup**:
- Output channels are properly closed using `sync.Once` to prevent double-closing.
- Process pipes are automatically cleaned up by Go's exec package.
- Memory resources are released when job is removed from tracking.

### **Error Handling**

**Common Scenarios**:
- **Job Not Found**: Returns error if job ID doesn't exist.
- **Job Already Stopped**: Returns error if job is not in `JOB_STATUS_RUNNING` status.
- **Process Kill Failure**: Returns error if OS-level process termination fails.
- **Network Errors**: Handles client disconnection during stop operation.

**Graceful Degradation**:
- If stop operation fails, job remains in `JOB_STATUS_RUNNING` status.
- Server continues to track the job until it naturally completes.
- No partial state corruption occurs.

### **Implementation Details**

**Thread Safety**:
- Job stopping is protected by mutex to prevent race conditions.
- Status updates are atomic and consistent.
- Multiple clients can attempt to stop the same job safely.

**Signal Handling**:
- Uses SIGKILL for immediate termination (no graceful shutdown for individual jobs).
- Process group is handled automatically by Go's exec package.
- No custom signal handling needed for job termination.

---
## gRPC Streaming Lifecycle Management

### **Stream Lifecycle**

**Stream Creation**: Client initiates `StreamJobOutput` RPC; server validates authorization, creates dedicated client channel, and adds it to the job's broadcast list.

**Stream Termination**: Streams end when job completes, client disconnects, server shuts down, or error occurs. Client channels are automatically removed from broadcast list and closed.

### **Client Disconnection Handling**

**Detection**: Monitor `stream.Context().Done()` for client cancellation and detect network errors during `stream.Send()`.

**Cleanup**: Remove client channel from job's broadcast list; close client channel; continue serving other clients; log disconnection.

### **Stream Patterns**

**Concurrent Streaming**: Multiple clients can stream same job output; each gets independent stream with dedicated client channel.

**Error Handling**: Network errors terminate stream; authorization failures return error; context cancellation triggers graceful cleanup.

**Implementation**:
```go
// Create dedicated channel for this client
clientChan := make(chan []byte)
defer job.RemoveClient(clientChan)

// Add to broadcast list
job.AddClient(clientChan)

for {
    select {
    case data, ok := <-clientChan:
        if !ok { return nil } // Job completed
        if err := stream.Send(data); err != nil { return err } // Client disconnected
    case <-ctx.Done():
        return ctx.Err() // Client cancelled
    }
}
```

### **Graceful Shutdown Handling**

**Server Shutdown**: During graceful shutdown, all jobs (running and completed) have their client channels closed via `CloseAllClients()` to ensure no goroutines are left waiting on closed channels.

**Channel Cleanup**: The `CleanupJobs()` method ensures all client channels are properly closed for both running jobs (via `Stop()`) and completed jobs (via `CloseAllClients()`).

---

## Security Considerations

- **mTLS**: All API communication uses mutual TLS. Only clients with valid certificates (signed by a trusted CA) can connect.
- **TLS Configuration**: Use TLS 1.3 (or 1.2 as fallback), strong cipher suites (Go will use it's default cipher suites for TLS 1.2+), and secure certificate/key handling.
- **Authorization**: Simple hardcoded authentication: only clients with certificate subjects in the hardcoded allowlist can access the API.
- **Input Validation**: All user input (commands, arguments) is validated and sanitized. EG: check for forbidden characters, length check etc.
- **Process Isolation**: Each job runs as a child process of the worker. No shell interpretation; arguments are passed directly to the executable.
- **Resource Limits**: Jobs can be placed in cgroups for resource control (CPU, memory) **(Out of scope)**.

### Security Implementation Details

- **CA Setup**: The Certificate Authority (CA) is generated for this project using Go's standard library cryptographic packages (`crypto/rsa`, `crypto/x509`, `crypto/rand`). The CA certificate, server certificate, and client certificates are created and distributed as part of the project setup. The CA certificate is stored in the `certs/` directory, and all components reference this path for trust.

  **Why Go's standard library instead of OpenSSL:**
  - **Portability**: No external dependencies required - works on any system with Go installed.
  - **Consistency**: Uses the same cryptographic libraries that the application uses for TLS.
  - **Security**: Go's crypto packages are well-audited and maintained as part of the standard library.
  - **Reproducibility**: Certificate generation is deterministic and version-controlled with the codebase.

- **mTLS Verification Implementation**:
  - **Server Configuration**: gRPC server uses `credentials.NewServerTLSFromFile()` to load server certificate and key.
  - **Client Verification**: Server requires client certificates via `grpc.Creds(creds)` with mutual TLS enabled.
  - **Role Extraction**: Client certificate subject (Distinguished Name) is extracted using `tlsInfo.State.PeerCertificates[0].Subject.String()`.
  - **Authentication Flow**: Certificate subject is compared against hardcoded allowlist; access granted only if subject matches.

- **Authentication Implementation**:
  - **Hardcoded Allowlist**: Client certificate subjects are hardcoded in the server with the following allowed users: `CN=client`, `CN=admin`, `CN=developer`, `CN=analyst`, `CN=readonly`.
  - **Authentication Check**: Server validates client certificate subject against the hardcoded allowlist before allowing access to any API.
  - **Security**: Only clients whose certificate subject matches an entry in the hardcoded allowlist are permitted to use the API.
  - **Simplicity**: No external file dependencies; all authentication rules are embedded in the server code.

- **Authorization Implementation**:
  - **Multiple User Profiles**: Hardcoded authorization profiles for different user types with varying command permissions.
  - **Admin Profile** (`CN=admin`): Full system access including `ps`, `top`, `df`, `du`, `who`, `w`.
  - **Developer Profile** (`CN=developer`): Development tools access including `git`, `go`, `make`.
  - **Read-only Profile** (`CN=readonly`): Limited to basic read operations like `echo`, `ls`, `cat`, `head`, `tail`.
  - **Default Profile**: Safe commands for users not explicitly configured.
  - **Authorization Check**: Server validates that the requested command is in the user's allowed command list before allowing job execution.
  - **Security**: Users can only execute commands they are explicitly authorized to run, preventing unauthorized system access.

- **TLS Version**:  
  The server is explicitly configured to accept only TLS 1.3 connections for the reasons below:
  - **Modern Security**: TLS 1.3 is the current recommended version and meets modern security requirements.
  - **Vulnerability Protection**: Eliminates known vulnerabilities like BEAST, Lucky 13, and POODLE attacks.
  - **Performance**: Faster handshake (one round trip) and reduced latency (especially important for gRPC streaming).
  - **Enforcement**: Both MinVersion and MaxVersion are set to TLS 1.3 to ensure only TLS 1.3 connections are accepted.

- **Certs and Security**:
  - Certificates and private keys are stored with strict file permissions (readable only by the service user).
  - Private keys are never committed to version control.
  - Authentication rules are hardcoded in the server and should be protected from tampering.
  - Certificates should be rotated regularly.
  - Access to the certs/ directory should be monitored and audited.

---

## Edge Cases & Error Handling

- **Process Crashes**: Status and output are available until explicitly cleaned up.
- **Output Buffering**: Output is available from process start, even if clients connect later.
- **Concurrent Access**: Multiple clients can stream output or query status concurrently.
- **Invalid Input**: All errors are reported clearly; invalid commands or unauthorized actions are rejected with descriptive messages.
- **Process Output Type**: The system handles both text and binary output without assumptions.
- **Network and Transport Errors**: The system gracefully handles network interruptions and TLS handshake failures.
- **Authorization and Authentication Failures**: Failed authentication (invalid/missing certificates) and failed authorization are handled gracefully and logged.
- **Zombie Processes**: The worker library is responsible for reaping child processes through `Cmd.Wait()` calls in goroutines, ensuring automatic cleanup when processes terminate.
- **Process Group Management**: Child processes are managed through Go's exec package, which handles process group creation and cleanup automatically.
- **Race Conditions**: UUID-based job identification prevents race conditions in concurrent job creation scenarios.

---

## Implementation Plan

### **PR Breakdown**

**PR #1: Design & Protocol Definition**
- Design document with complete architecture specification
- Protocol buffer definitions (`job_worker.proto`)
- Project structure and documentation

**PR #2: Security Infrastructure**
- Certificate generation utility with Go standard library
- TLS configuration with mTLS and strong cipher suites
- Hardcoded authentication and authorization implementation
- Input validation and sanitization

**PR #3: Worker Library**
- Job management with UUID-based identification
- Process creation and cleanup
- Output streaming architecture (Direct Pipe with Multiplexing)
- Channel-based output buffering and multiplexing

**PR #4: gRPC Server**
- Server implementation with job operations (start, stop, status)
- gRPC streaming for job output
- Integration with worker library
- Error handling and graceful shutdown

**PR #5: CLI Client**
- Command-line interface with mTLS support
- Job management commands (start, stop, status, stream)
- Real-time output streaming to terminal
- Error handling and user experience

### **Testing Approach**

**Unit Tests**
- **Worker Library**: Test job creation, start/stop, output capture, and cleanup.
- **Authorization**: Test hardcoded authentication validation with valid/invalid certificates.
- **Input Validation**: Test command/argument sanitization and length limits.

**Integration Tests**
- **mTLS Communication**: Test client-server authentication with valid/invalid certs.
- **Output Streaming**: Test real-time output capture and multi-client streaming.
- **Process Management**: Test child process cleanup and zombie prevention.

### **Output Guarantee from Start**

**Technical Implementation**
- Output capture begins **before** `Cmd.Start()` by setting up pipes first.
- `StdoutPipe()` and `StderrPipe()` are created and connected to the child process.
- Goroutines start reading from pipes immediately when process begins.
- No output is lost because pipes are established before process execution.

**Verification Method**
- Test with commands that produce immediate output (e.g., `echo "hello"`).
- Verify first byte appears in stream before process completion.
- Measure latency from process start to first output byte.
- Test with high-frequency output generators.

---

## Conclusion

This design provides a secure, efficient, and extensible foundation for a job worker service, with a focus on robust process management, secure communication, and a clear CLI user experience. Feedback is welcome.
