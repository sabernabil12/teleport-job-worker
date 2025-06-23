# Job Worker Service: Design Document

## Table of Contents

- [Job Worker Service: Design Document](#job-worker-service-design-document)
  - [Table of Contents](#table-of-contents)
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
      - [Messages](#messages)
  - [Design Approach](#design-approach)
  - [Graceful Shutdown](#graceful-shutdown)
    - [Shutdown Sequence](#shutdown-sequence)
    - [Job Cleanup](#job-cleanup)
    - [Timeout Management](#timeout-management)
    - [Implementation Details](#implementation-details)
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
  - [gRPC Streaming Lifecycle Management](#grpc-streaming-lifecycle-management)
    - [**Stream Lifecycle**](#stream-lifecycle)
    - [**Client Disconnection Handling**](#client-disconnection-handling)
    - [**Stream Patterns**](#stream-patterns)
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
- **Strong Cipher Suite**: Explicitly configured `TLS_AES_256_GCM_SHA384` for maximum security.
- **File-based Authorization**: Client certificate subjects validated against allowlist.
- **Input Validation**: Comprehensive command and argument sanitization.
- **Process Isolation**: Jobs run as isolated child processes without shell interpretation.

#### **Performance & Reliability**
- **Real-time Output Streaming**: Live job output via gRPC streaming with proper lifecycle management.
- **Concurrent Job Support**: Multiple jobs and clients handled simultaneously.
- **UUID-based Job IDs**: Prevents race conditions and ensures uniqueness.
- **Graceful Shutdown**: Signal handling with job cleanup and 30-second timeout.
- **Memory-efficient Buffering**: 100-item buffer per job prevents unbounded memory growth.

#### **User Experience**
- **Simple CLI Interface**: Intuitive commands without complex flags.
- **Real-time Output**: Job output streamed to terminal immediately.
- **Comprehensive Status**: Job status, exit codes, start/end times.
- **Cross-platform**: Works on Linux, macOS, and other Unix-like systems.

#### **Architecture & Design**
- **Direct Pipe with Multiplexing**: Efficient output streaming architecture.
- **Zero-copy Data Flow**: Direct from process pipes to network streams.
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

#### Messages

- `StartJobRequest`: Command (string), arguments ([]string), environment (map[string]string) - TODO: Environment variable support not yet implemented in server.
- `StartJobResponse`: Job ID (string), initial status (string), start time (string).
- `StopJobRequest`: Job ID (string).
- `StopJobResponse`: Success/failure (bool), end time (string).
- `GetJobStatusRequest`: Job ID (string).
- `GetJobStatusResponse`: Status (running, completed, error, stopped), exit code (int32), start time (string), end time (string).
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

---

## Implementation Details

### Worker Library

- **Job Management**: Maintains a map of job UUIDs to process handles and metadata. Uses atomic UUID generation to avoid race conditions in concurrent job creation.
- **Process Management**: Each job creates a child process using `exec.Command()`.
- **Process Reaping**: For every job, a dedicated goroutine calls `Cmd.Wait()` as soon as the process exits, ensuring all child processes are properly reaped and no zombies remain.
- **Output Streaming**: Implements Direct Pipe with Multiplexing architecture (see [Output Streaming Architecture](#output-streaming-architecture) for detailed technical implementation). Uses `os.Pipe` to capture stdout and stderr from child processes. Both stdout and stderr are combined into a single output channel that streams raw bytes without assumptions about content type (text/binary). Output is buffered in channels (100-item buffer per job) and can be streamed to multiple clients from the start. Output is captured using goroutines that read from process pipes and write to channels, enabling real-time streaming without polling or busy-waiting. The capture starts immediately when the process begins, ensuring no output is lost.
- **Concurrency**: Uses goroutines and channels for process management and output streaming.

### API Server

- **gRPC**: Exposes job management APIs.
- **TLS**: Configured with strong cipher suites, client cert verification, and secure key storage.
- **Authorization**: Implements a simple authorization scheme by checking the client certificate subject against an allowlist before allowing access to any API. Only clients whose certificate subject matches an entry in the allowlist are permitted to use the API.

### CLI

- **User Experience**: Simple, consistent commands for job management.
- **mTLS**: Requires client cert/key and CA cert for all operations.
- **Output Streaming**: Streams output to stdout/stderr in real time.

---

## Output Streaming Architecture

The system implements **Direct Pipe with Multiplexing** for efficient real-time output streaming. Here's the concrete architecture:

### **Implementation Details**

1. **Process Output Capture**:
   - Each job creates child process using `exec.Command()`.
   - `StdoutPipe()` and `StderrPipe()` create OS-level pipes connected to the child process.
   - Two goroutines run `io.Copy(&channelWriter{job.Output}, pipe)` for stdout and stderr.
   - Both stdout and stderr are combined into a single output channel.
   - Raw bytes are immediately written to a buffered channel (100-item capacity per job).

2. **Channel-Based Multiplexing**:
   - Each job has a dedicated `chan []byte` with 100-item buffer.
   - `channelWriter` implements `io.Writer` interface to bridge pipes to channels.
   - Multiple gRPC clients can read from the same channel concurrently.
   - Channel is closed when process terminates to signal end-of-stream.

3. **gRPC Streaming Layer**:
   - Server maintains a map of job UUIDs to worker.Job instances.
   - `StreamJobOutput` RPC creates a goroutine that monitors job status.
   - Main loop uses `select` to handle: output data, context cancellation, or job completion.
   - Each client gets a dedicated gRPC stream that reads from the shared job output channel.

### **Data Flow**
```
Child Process (stdout + stderr) 
    ↓ (OS pipes)
io.Copy() goroutines (2x)
    ↓ (channelWriter)
Single Buffered Channel (100 items)
    ↓ (multiple readers)
gRPC Stream Clients
```

### **Key Benefits**
- **Zero-copy**: Data flows directly from process pipes to network streams.
- **Real-time**: Output appears immediately as process produces it.
- **Concurrent**: Multiple clients can stream the same job output.
- **Memory efficient**: 100-item buffer prevents unbounded memory growth.
- **Automatic cleanup**: Channel closure signals end-of-stream to all clients.
- **Simplified**: Single output stream eliminates complexity of separate stdout/stderr handling.

### **Why Not Other Approaches**
- **File + fsnotify**: Would require disk I/O and file management overhead.
- **Ring buffer**: More complex, doesn't provide the same real-time guarantees.
- **Separate stdout/stderr channels**: Added complexity without significant benefit for most use cases.

### **Why this approach is efficient**
- **Uses blocking I/O:** Goroutines read from OS pipes and sleep until new output is available, consuming no CPU while idle.
- **No polling or busy-waiting:** The system reacts instantly to new data, rather than repeatedly checking for it.
- **Real-time delivery:** Output is pushed to clients as soon as it is produced, ensuring low latency.
- **Resource efficient:** CPU and memory usage remain minimal, as work only occurs when there is actual process output.

---

## gRPC Streaming Lifecycle Management

### **Stream Lifecycle**

**Stream Creation**: Client initiates `StreamJobOutput` RPC; server validates authorization and creates dedicated goroutine for stream management.

**Stream Termination**: Streams end when job completes, client disconnects, server shuts down, or error occurs.

### **Client Disconnection Handling**

**Detection**: Monitor `stream.Context().Done()` for client cancellation and detect network errors during `stream.Send()`.

**Cleanup**: Stop reading from job output channel for disconnected client; continue serving other clients; log disconnection.

### **Stream Patterns**

**Concurrent Streaming**: Multiple clients can stream same job output; each gets independent stream with shared job output channel.

**Error Handling**: Network errors terminate stream; authorization failures return error; context cancellation triggers graceful cleanup.

**Implementation**:
```go
for {
    select {
    case data, ok := <-jobOutput:
        if !ok { return nil } // Job completed
        if err := stream.Send(data); err != nil { return err } // Client disconnected
    case <-ctx.Done():
        return ctx.Err() // Client cancelled
    }
}
```

---

## Security Considerations

- **mTLS**: All API communication uses mutual TLS. Only clients with valid certificates (signed by a trusted CA) can connect.
- **TLS Configuration**: Use TLS 1.3 (or 1.2 as fallback), strong cipher suites (Go will use it's default cipher suites for TLS 1.2+), and secure certificate/key handling.
- **Authorization**: Simple allowlist: only clients with certificate subjects in the allowlist can access the API.
- **Input Validation**: All user input (commands, arguments, environment) is validated and sanitized. EG: check for forbidden characters, length check etc. 
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
  - **Authorization Flow**: Certificate subject is compared against hardcoded allowlist in server code; access granted only if subject matches.

- **Authorization Implementation**:
  - **File-Based Allowlist**: Client certificate subjects are loaded from `certs/allowlist.txt` at server startup.
  - **Allowlist Format**: One certificate subject per line, comments start with #, empty lines ignored.
  - **Current Allowlist**: `"CN=client"` - matches the client certificate generated by the certificate utility.
  - **Authorization Check**: Server validates client certificate subject against the loaded allowlist before allowing access to any API.
  - **Security**: Only clients whose certificate subject matches an entry in the allowlist are permitted to use the API.
  - **Reload**: Allowlist is loaded at startup; server restart required for changes to take effect.

- **TLS Version**:  
  The server is explicitly configured to accept only TLS 1.3 connections for the reasons below:
  - **Modern Security**: TLS 1.3 is the current recommended version and meets modern security requirements.
  - **Vulnerability Protection**: Eliminates known vulnerabilities like BEAST, Lucky 13, and POODLE attacks.
  - **Performance**: Faster handshake (one round trip) and reduced latency (especially important for gRPC streaming).
  - **Enforcement**: Both MinVersion and MaxVersion are set to TLS 1.3 to ensure only TLS 1.3 connections are accepted.

- **Cipher Suites**:
  Explicitly configured to use `TLS_AES_256_GCM_SHA384` (over Go's defaults) as the preferred cipher suite. This choice is made because:
  - **Maximum Security**: Ensures we get the strongest available cipher suite rather than potentially weaker defaults.
  - **Predictable Security**: No risk of Go's defaults changing in future releases to include less secure options.
  - **Explicit Control**: We know exactly what cryptographic algorithms are being used.
  - **Future-Proof**: 256-bit AES with GCM mode represents the current gold standard for TLS security.
  - **Compliance**: Meets strict security requirements for job execution systems where security is paramount.

- **Certs and Allowlist Security**:
  - Certificates and private keys are stored with strict file permissions (readable only by the service user).
  - Private keys are never committed to version control.
  - The allowlist file (`certs/allowlist.txt`) is stored with restricted permissions (readable only by the server process).
  - The allowlist file should be protected from tampering and monitored for unauthorized changes.
  - Certificates and allowlist entries should be rotated regularly.
  - Access to the certs/ directory and allowlist file should be monitored and audited.

---

## Edge Cases & Error Handling

- **Process Crashes**: Status and output are available until explicitly cleaned up.
- **Output Buffering**: Output is available from process start, even if clients connect later.
- **Concurrent Access**: Multiple clients can stream output or query status concurrently.
- **Invalid Input**: All errors are reported clearly; invalid commands or unauthorized actions are rejected with descriptive messages.
- **Process Output Type**: The system handles both text and binary output without assumptions.
- **Network and Transport Errors**: The system gracefully handles network interruptions and TLS handshake failures.
- **Authorization and Authentication Failures**: Failed authentication (invalid/missing certificates) and failed authorization (not on allowlist) are handled gracefully and logged.
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
- Authorization allowlist implementation
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
- **Authorization**: Test allowlist validation with valid/invalid certificates.
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