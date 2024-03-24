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

Please see [job_worker.proto](/proto/job_worker.proto) for the full protobuf specification.

## CLI UX

The CLI will provide commands that correspond to the API endpoints. Here are some example usages:

- `job-worker server start --address localhost:50051 --cert /path/to/cert --key /path/to/key`: Start the server listening on `localhost:50051` with the specified certificate and key.
- `job-worker client start "ls -la"`: Start a new job with the command `ls -la`.
- `job-worker client status 550e8400-e29b-41d4-a716-446655440000`: Get the status of the job with ID `550e8400-e29b-41d4-a716-446655440000`.
- `job-worker client logs 550e8400-e29b-41d4-a716-446655440000`: Get the output of the job with ID `550e8400-e29b-41d4-a716-446655440000`.
- `job-worker client stop 550e8400-e29b-41d4-a716-446655440000`: Stop the job with ID `550e8400-e29b-41d4-a716-446655440000`.

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

The `Logs` API method will stream the output (both stdout and stderr) of a job in real-time. This will be achieved by using gRPC's server-side streaming capabilities. When a job is started, the server will continuously send `JobLogsResponse` messages to the client over a single RPC as the output of the job becomes available. The client can read these messages as they arrive, providing a real-time stream of logs.

Within the framework of gRPC streaming, the design of the logging system is intended to facilitate simultaneous output streaming by multiple clients. This is accomplished by writing the output to a file on disk, and tracking each client's read position.

1. Upon initiation of a job, the output is captured and written into an `os.File` on disk. This is achieved by employing the `os/exec` package's `Cmd.StdoutPipe` and `Cmd.StderrPipe` methods. These methods return `io.ReadCloser` instances that can be read to capture the command's output.

2. When a client requests to stream the job's output, a new `Subscriber` is created. This `Subscriber` holds the gRPC stream for the client and the offset up to which the client has read the file. The `Subscriber` is added to a map of subscribers maintained by the job, with the key being the unique identifier for the subscription (not the client ID, as a single client can have multiple subscriptions). Access to this map is synchronized using a mutex to ensure thread-safety.

3. A loop is started that continues until the job is complete. In each iteration of the loop:
   - A mutex is locked to ensure exclusive access to the job and file.
   - The current size of the file is checked. If it's larger than a subscriber's current offset, it means there's data the subscriber hasn't read yet.
   - The file is seeked to the subscriber's offset.
   - Lines are read from the file until the end of the currently available data is reached.
   - For each line, the subscriber's offset is updated and the line is sent to the client over the gRPC stream.
   - If all currently available data has been read and the job is still running, the mutex is unlocked, there's a short wait (e.g., 100ms), and the next iteration starts.

4. When a client disconnects or the job completes, the `Subscriber` is removed from the job's map of subscribers.

This approach ensures that each client receives the full output of the job, starting from the beginning, regardless of when they start streaming. Each client has its own offset into the file, so clients that start streaming later will first receive the earlier parts of the output, and then catch up to the live output.

The use of a mutex ensures that access to the shared job and file resources is synchronized across all clients.

## Implementation Details

The library will be implemented in Go, using the `os/exec` package for process management. The API server will be implemented using the gRPC framework. The CLI will be a simple Go application that makes gRPC calls to the API server.

Job IDs will be generated using a universally unique identifier (UUID). This will be done using the `github.com/google/uuid` package.

When a new job is started, the library will create a new `exec.Cmd` instance, start the command, and add the command to a map of running jobs. The command's output will be captured in a shared buffer, and will be sent to the client over the gRPC stream.

To stop a job, the library will use the `Cmd.Process.Kill` method. This will send a SIGKILL signal to the process, causing it to terminate immediately.

Resource control for CPU, Memory and Disk IO per job will be implemented using cgroups v2. When a new job is started, the library will create a new cgroup for the job, set the resource limits, and add the job's process to the cgroup immediately upon its start. This ensures that the resource limits are enforced from the very beginning of the process's execution. To ensure that the process is added to the cgroup immediately upon start, we use the `os/exec` package in Go to start the process and get its PID. Then, we add the PID to the cgroup. To further ensure that the process and all of its child processes remain within their assigned cgroup and cannot escape, we will use cgroup namespaces. A new cgroup namespace will be created for each job. This namespace will isolate the view of the cgroup hierarchy for the job's process, meaning it can only see and interact with its own cgroup.

The specific resource limits that will be enforced are:

1. CPU: The `cpu.max` parameter will be set to control the total amount of CPU time that the job's processes can consume.

2. Memory: The `memory.max` parameter will be set to control the maximum amount of memory that the job's processes can use.

3. IO: The `io.max` parameter will be set to control the maximum amount of read and write operations that the job's processes can perform.

The mTLS authentication will be implemented using Go's `crypto/tls` package. The server will require client certificates and verify them against a trusted certificate authority. The client will also verify the server's certificate.

The authorization scheme will be simple: each job will be associated with the client that started it, and clients will only be allowed to control their own jobs. This will be enforced by the API server.

The CLI will parse command-line arguments and make the appropriate gRPC calls to the API server. It will print the responses to the console.

The `Logs` method will be implemented as follows: When a client calls `Logs`, a new `Subscriber` will be created and added to the job's list of subscribers. A loop will be started that continues until the job is complete. In each iteration, a mutex will be locked, the file size will be checked, and if there's new data for the subscriber, it will be read and sent over the gRPC stream. The mutex will be unlocked, and if the job is still running, there will be a short wait before the next iteration. When the client disconnects or the job completes, the `Subscriber` will be removed from the job's map of subscribers.

## Performance Considerations

The current design checks the file size in each iteration of the loop. While this approach is simple and effective, it might not be the most efficient if the file is very large or updates are infrequent. Future optimizations could include using a file change notification system, if available on the platform, or adjusting the polling frequency based on the rate of updates.

## Testing

Unit tests will be written for the library, API server, and CLI. The tests will cover both happy and unhappy scenarios. For example, starting and stopping a job, trying to stop a job that doesn't exist, trying to stop a job that was started by another client, etc.

In addition to these functional tests, we will also include tests for authentication, authorization, and logging:

- Authentication tests will verify that only clients with valid certificates can connect to the API server.
- Authorization tests will ensure that clients can only control jobs that they own.
- Logging tests will check that all important events are logged correctly, and that logs contain all necessary information.