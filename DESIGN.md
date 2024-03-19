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
```

## Security Strategy

The API will leverage mTLS authentication and validate client certificates. A robust set of cipher suites for TLS and a secure crypto setup for certificates will be employed. mTLS will be the sole authentication protocol. A basic authorization mechanism will be implemented, where each client is permitted to control only its own jobs.

## CLI UX

The CLI will provide commands that correspond to the API endpoints. Here are some example usages:

- `job-worker start "ls -la"`: Start a new job with the command `ls -la`.
- `job-worker status 123`: Get the status of the job with ID 123.
- `job-worker logs 123`: Get the output of the job with ID 123.
- `job-worker stop 123`: Stop the job with ID 123.

## Streaming Strategy

The `Logs` API method will stream the output of a job in real-time. This will be achieved by using a WebSocket connection between the client and the server. When a job is started, a new WebSocket connection will be opened. The server will then send the output of the job over this connection as it becomes available.

## TLS Setup

The API server will use mTLS for secure communication. The server will be configured to use TLS 1.3, which provides improved security and performance compared to previous versions. The server will also be configured to use secure cipher suites such as ECDHE-ECDSA-AES128-GCM-SHA256 and ECDHE-RSA-AES128-GCM-SHA256.

## Process Execution Lifecycle

The library will use the `os/exec` package to start and stop jobs. When a new job is started, the library will create a new `exec.Cmd` instance, start the command, and add the command to a map of running jobs. The command's output will be captured in a buffer, and will be sent to the client over the WebSocket connection.

To stop a job, the library will use the `Cmd.Process.Kill` method. This will send a SIGKILL signal to the process, causing it to terminate immediately.

The library will also use cgroups to limit the resources that a job can use. When a new job is started, the library will create a new cgroup for the job, set the resource limits, and add the job's process to the cgroup.

## Implementation Details

The library will be implemented in Go, using the `os/exec` package for process management. The API server will be implemented using the gRPC framework. The CLI will be a simple Go application that makes gRPC calls to the API server.

Resource control for CPU, Memory and Disk IO per job will be implemented using cgroups. The library will create a new cgroup for each job, set the resource limits, and add the process to the cgroup.

The mTLS authentication will be implemented using Go's `crypto/tls` package. The server will require client certificates and verify them against a trusted certificate authority. The client will also verify the server's certificate.

The authorization scheme will be simple: each job will be associated with the client that started it, and clients will only be allowed to control their own jobs. This will be enforced by the API server.

The CLI will parse command-line arguments and make the appropriate gRPC calls to the API server. It will print the responses to the console.

## Testing

Unit tests will be written for the library, API server, and CLI. The tests will cover both happy and unhappy scenarios. For example, starting and stopping a job, trying to stop a job that doesn't exist, trying to stop a job that was started by another client, etc.
