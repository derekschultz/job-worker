# Job Worker Service Design Document

## Design Approach

The job worker service will be composed of three main components:

1. A library that provides the core functionality of running and managing Linux processes.
2. An API server that exposes the library's functionality over a network.
3. A command-line interface (CLI) that interacts with the API server to manage processes.

The library will be constructed with a focus on process management. The design of the API server and CLI will prioritize user-friendliness, security, and robustness.

## Scope

The scope of this project includes the design and implementation of the library, API server, and CLI. It also includes the setup of a secure communication channel between the CLI and the API server, and the implementation of a simple authorization scheme.

## Proposed API

The API will be a gRPC API with the following methods:

- `Start`: Starts a new job.
- `Stop`: Stops an existing job.
- `Status`: Gets the status of a job.
- `Logs`: Streams the output of a job.

## Protobuf Specification

```protobuf
service JobWorker {
  rpc Start(JobRequest) returns (JobResponse) {}
  rpc Stop(JobRequest) returns (JobResponse) {}
  rpc Status(JobRequest) returns (JobStatusResponse) {}
  rpc Logs(JobRequest) returns (stream JobLogsResponse) {}
}

message JobRequest {
  string id = 1;  // unique identifier of the job
}

message JobResponse {
  string id = 1;
  string status = 2;  // status of the job (e.g., "running", "stopped")
}

message JobStatusResponse {
  string id = 1;
  string status = 2;
  string output = 3;
}

message JobLogsResponse {
  string log = 1;
}
```

## CLI UX

The CLI will provide commands that correspond to the API endpoints. Here are some example usages:

- `job-worker start "ls -la"`: Start a new job with the command `ls -la`.
- `job-worker status 550e8400-e29b-41d4-a716-446655440000`: Get the status of the job with ID `550e8400-e29b-41d4-a716-446655440000`.
- `job-worker logs 550e8400-e29b-41d4-a716-446655440000`: Get the output of the job with ID `550e8400-e29b-41d4-a716-446655440000`.
- `job-worker stop 550e8400-e29b-41d4-a716-446655440000`: Stop the job with ID `550e8400-e29b-41d4-a716-446655440000`.

## Security Strategy

A basic authorization mechanism will be implemented, where each client is permitted to control only its own jobs. This will be enforced by checking the client's ID against the owner of the job in each request.

The client will be identified by the Common Name (CN) field in the certificate it presents to the server during the TLS handshake. The server will map this CN to a specific client ID in its internal database.

The authorization scheme will be defined as follows:

```go
func (s *Server) authorizeRequest(request *JobRequest, clientID string) error {
    job, ok := s.Jobs[request.JobID]
    if !ok {
        return fmt.Errorf("job not found")
    }
    if job.Owner != clientID {
        return fmt.Errorf("permission denied")
    }
    return nil
}
```

The API server will use mTLS for secure communication, specifically using TLS 1.3. It's important to note that in the context of Go and TLS 1.3, cipher suites are not subject to configuration. This is due to the design of TLS 1.3, which inherently ensures the security of the connection by using a set of predefined, secure cipher suites.

## Streaming Strategy

The `Logs` API method will stream the output of a job in real-time. This will be achieved by using gRPC's server-side streaming capabilities. When a job is started, the server will continuously send `JobLogsResponse` messages to the client over a single RPC as the output of the job becomes available. The client can read these messages as they arrive, providing a real-time stream of logs.

Within the framework of gRPC streaming, the design of the logging system is intended to facilitate simultaneous output streaming by multiple clients. This is accomplished by utilizing a shared circular buffer stored in memory, and a mechanism for tracking each client's read position.

1. Upon initiation of a job, the output is captured and written into a shared buffer in memory. This is achieved by employing the `os/exec` package's `Cmd.StdoutPipe` and `Cmd.StderrPipe` methods. These methods return `io.ReadCloser` instances that can be read to capture the command's output.

2. Each client that requests to stream the job's output is provided with a reader that commences at its current read position in the buffer. This is facilitated by maintaining a map that links client IDs to read positions.

3. As the job generates output, it is written into the shared circular buffer in memory. If the buffer is full, new data starts overwriting the oldest data in the buffer. Each client's reader then reads from the buffer, starting at its own position. This enables each client to receive the job's output in real-time, irrespective of the timing of their streaming initiation.

4. In the event of a client disconnecting or ceasing to stream the output, the associated reader and position can be discarded.

## Implementation Details

The library will be implemented in Go, using the `os/exec` package for process management. The API server will be implemented using the gRPC framework. The CLI will be a simple Go application that makes gRPC calls to the API server.

Job IDs will be generated using a universally unique identifier (UUID). This will be done using the `github.com/google/uuid` package.

When a new job is started, the library will create a new `exec.Cmd` instance, start the command, and add the command to a map of running jobs. The command's output will be captured in a shared buffer, and will be sent to the client over the gRPC stream.

To stop a job, the library will use the `Cmd.Process.Kill` method. This will send a SIGKILL signal to the process, causing it to terminate immediately.

Resource control for CPU, Memory and Disk IO per job will be implemented using cgroups v2. When a new job is started, the library will create a new cgroup for the job, set the resource limits (such as CPU usage, memory usage, and disk I/O), and add the job's process to the cgroup immediately upon its start. This ensures that the resource limits are enforced from the very beginning of the process's execution.

The mTLS authentication will be implemented using Go's `crypto/tls` package. The server will require client certificates and verify them against a trusted certificate authority. The client will also verify the server's certificate.

The authorization scheme will be simple: each job will be associated with the client that started it, and clients will only be allowed to control their own jobs. This will be enforced by the API server.

The CLI will parse command-line arguments and make the appropriate gRPC calls to the API server. It will print the responses to the console.

## Testing

Unit tests will be written for the library, API server, and CLI. The tests will cover both happy and unhappy scenarios. For example, starting and stopping a job, trying to stop a job that doesn't exist, trying to stop a job that was started by another client, etc.

In addition to these functional tests, we will also include tests for authentication, authorization, and logging:

- Authentication tests will verify that only clients with valid certificates can connect to the API server.
- Authorization tests will ensure that clients can only control jobs that they own.
- Logging tests will check that all important events are logged correctly, and that logs contain all necessary information.