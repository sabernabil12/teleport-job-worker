# Teleport Job Worker

A gRPC-based job worker system with secure authentication.

## Project Structure

```
teleport-job-worker/
├── cmd/                    # Application entry points
│   ├── cli/               # Command-line interface
│   ├── server/            # Server application
│   └── generate-certs/    # Certificate generation utility
├── internal/              # Internal packages (not importable externally)
│   ├── server/            # Server implementation
│   └── worker/            # Worker implementation
├── pkg/                   # Public packages (importable by other projects)
│   └── proto/             # Protocol buffer definitions and generated code
├── tools/                 # Development and build tools
├── build/                 # Build artifacts (auto-generated)
├── certs/                 # Certificate directory (auto-generated)
├── docs/                  # Documentation
│   ├── README.md          # This file
│   └── design-doc.md      # Project design documentation
├── go.mod                 # Go module file
└── go.sum                 # Go module checksums
```

## Prerequisites

- **Go 1.24+**: Required for building and running the applications
- **Protocol Buffers**: Required for generating gRPC code (optional, pre-generated code included)

## Quick Start

```bash
# 1. Generate certificates
go run cmd/generate-certs/main.go

# 2. Build applications
go build -o build/server cmd/server/main.go
go build -o build/cli cmd/cli/main.go

# 3. Start server (in one terminal)
./build/server

# 4. Use CLI (in another terminal)
./build/cli start echo "Hello World"
```

## Key Features

- Secure mTLS Communication: TLS 1.3 with client certificate verification
- Process Isolation: Jobs run as isolated child processes without shell interpretation
- Real-time Output Streaming: Live job output via gRPC streaming
- Input Validation: Comprehensive command and argument sanitization
- Authorization: File-based allowlist for client certificate subjects
- Graceful Shutdown: Signal handling with job cleanup and timeout
- UUID-based Job IDs: Prevents race conditions and ensures uniqueness

## Setup

1. **Generate certificates and allowlist** (required for secure communication):
   ```bash
   go run cmd/generate-certs/main.go
   ```
   Or build and run the certificate generation tool:
   ```bash
   go build -o build/generate-certs cmd/generate-certs/main.go
   ./build/generate-certs
   ```
   This creates:
   - CA certificate and key (`certs/ca.crt`, `certs/ca.key`)
   - Server certificate and key (`certs/server.crt`, `certs/server.key`)
   - Client certificate and key (`certs/client.crt`, `certs/client.key`)
   - Allowlist file (`certs/allowlist.txt`) with the client certificate subject

2. **Build the applications**:
   ```bash
   # Build server
   go build -o build/server cmd/server/main.go
   
   # Build CLI client
   go build -o build/cli cmd/cli/main.go
   
   # Build certificate generation tool
   go build -o build/generate-certs cmd/generate-certs/main.go
   ```

3. **Run the server**:
   ```bash
   ./build/server
   ```

4. **Use the CLI client**:
   ```bash
   ./build/cli
   ```

### CLI Usage

The CLI client supports the following commands:

- `start <command> [args...]` - Start a new job
- `stop <job-id>` - Stop a running job
- `status <job-id>` - Get job status and details
- `stream <job-id>` - Stream real-time job output

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

**Note**: Job IDs are now UUIDs (e.g., `550e8400-e29b-41d4-a716-446655440000`) instead of incremental IDs to avoid race conditions in concurrent environments.

## Graceful Shutdown

The server implements graceful shutdown to ensure data integrity and proper resource cleanup. When shutting down, the server will:

1. Stop accepting new connections
2. Allow existing requests to complete (30-second timeout)
3. Stop all running jobs gracefully
4. Clean up resources and exit

### Shutdown Methods

**Method 1: Interactive shutdown (Ctrl+C)**
```bash
# Start the server
./build/server

# Press Ctrl+C to initiate graceful shutdown
# Server will log: "Received shutdown signal, starting graceful shutdown..."
```

**Method 2: Programmatic shutdown (SIGTERM)**
```bash
# Start the server in background
./build/server &
SERVER_PID=$!

# Send SIGTERM for graceful shutdown
kill $SERVER_PID

# Or find and kill by process name
pkill -f "build/server"
```

## Troubleshooting

### Common Issues

**Certificate errors:**
```bash
# Regenerate certificates if you encounter TLS errors
go run cmd/generate-certs/main.go
```

**Permission denied errors:**
```bash
# Ensure certificate files have correct permissions
chmod 600 certs/*.key certs/allowlist.txt
chmod 644 certs/*.crt
```

**Job not found errors:**
- Job IDs are UUIDs, make sure you're using the correct format
- Jobs are automatically cleaned up when the server restarts

**Connection refused:**
- Ensure the server is running on the expected port (default: 50051)
- Check that certificates are generated and in the correct location

## Testing

### Manual Testing

1. **Start the server**:
   ```bash
   ./build/server
   ```

2. **Test basic functionality**:
   ```bash
   # Start a simple job
   ./build/cli start echo "Hello World"
   
   # Start a longer-running job
   ./build/cli start sleep 10
   
   # Get job status
   ./build/cli status <job-id>
   
   # Stream output
   ./build/cli stream <job-id>
   
   # Stop a job
   ./build/cli stop <job-id>
   ```

3. **Test graceful shutdown**:
   - Start the server
   - Start some jobs
   - Press Ctrl+C to test graceful shutdown
   - Verify jobs are cleaned up properly

### Security Testing

1. **Test invalid certificates**:
   - Try connecting with wrong client certificate
   - Verify connection is rejected

2. **Test unauthorized commands**:
   - Try commands with forbidden characters
   - Verify they are rejected with clear error messages

## Development

- The project uses gRPC for communication between client and server
- TLS certificates are used for secure authentication
- The `certs/` directory contains generated certificates (not committed to version control)
- Build artifacts are placed in the `build/` directory
- Internal packages are in `internal/` and cannot be imported by external projects
- Public packages are in `pkg/` and can be imported by other projects
- Protocol buffers are defined in `pkg/proto/` and auto-generated Go code is included

## Security

- **TLS 1.3**: All communication is secured with TLS 1.3 (no fallback to older versions)
- **Strong Cipher Suite**: Uses `TLS_AES_256_GCM_SHA384` for maximum security
- **mTLS Authentication**: Client certificates are required for all connections
- **File-based Authorization**: Client certificate subjects are validated against `certs/allowlist.txt`
- **Input Validation**: All commands and arguments are sanitized and validated
- **Process Isolation**: Jobs run as isolated child processes without shell interpretation
- **Certificate Management**: Certificates are generated locally and should not be shared
- **File Permissions**: Private keys and allowlist have restricted permissions (600)

## Contributing

### Development Setup

1. **Fork and clone the repository**
2. **Install dependencies**:
   ```bash
   go mod download
   ```
3. **Generate certificates**:
   ```bash
   go run cmd/generate-certs/main.go
   ```
4. **Build the project**:
   ```bash
   go build -o build/server cmd/server/main.go
   go build -o build/cli cmd/cli/main.go
   ```

### Code Style

- Follow Go conventions and use `gofmt` for formatting
- Add tests for new features
- Update documentation for any API changes
- Use meaningful commit messages

### Project Structure

- **`cmd/`**: Application entry points
- **`internal/`**: Private packages (not importable externally)
- **`pkg/`**: Public packages (importable by other projects)
- **`docs/`**: Documentation and design documents

### Protocol Buffer Changes

If you modify `pkg/proto/job_worker.proto`:
```bash
protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative pkg/proto/job_worker.proto
``` 
