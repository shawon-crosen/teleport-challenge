# Design Document

Goal: Create an API to run arbitrary linux processes on a host utilizing a CLI
High Level Overview

The proposed solution will be divided into three parts:
1. The server side library written in Go to interact with the Linux OS
2. The gRPC API to take in requests from the client, use the server side library to run jobs, and return responses to the client
3. The CLI that the client will use to send requests related to jobs to the API

## Scope

### In Scope
* Starting a process
* Terminating a process
* Getting a process’s status
* Streaming the output of a process 

### Not in scope
* Listing jobs
* Stopping and restarting processes
* Task resource management (CPU, memory, etc)
* Automated certificate creation

## Approach
The current approach for this project is to create something as simple and straightforward as possible. This means that I will not be using any external libraries or dependencies, and will try to leverage the Linux operating system’s existing tooling as much as possible instead of trying to reinvent the wheel. 

This design will cut a lot of corners and leave a lot of ideal functionality out of the initial implementation. I have noted things that should be considered as `Future Improvements` in each section.

## API
The API will use gRPC to route requests from the client to the server. The API will run locally on the machine that is being managed from the client. We will have a single user on the machine we are running processes on named `processrunner`.

The API will accept POST requests for running processes and GET requests to return the status and output of a job. More detailed implementation details for the API can be found in the .proto document.

At a high level the API will have the following methods on the localhost using port 9090:

### Run/Stop Methods
`RunProcess(RunProcessRequest)`
`StopProcess(StopProcessRequest)`

### Get Information Methods
`GetProcessStatus(ProcessStatusRequest)`
`GetProcessOutput(ProcessOutputRequest)`

The POST to run the command will return the process identifier for the command issued to the client, which will be used for the GET requests for status and output.

### Future Improvements
* Run the server in a separate environment, such as kubernetes, which can allow for defining high availability through replicaSets and would move the API out from the domain of the operating system being managed
* Utilize some kind of data store (relational or not) to track processes instead of relying on the pid
* Include resource management of some kind to keep clients from over-taxing the operating system commands are running on
* Scope processes to users that map back to the clients on the operating system

## Server Authentication

### mTLS
The service will utilize mTLS to authenticate and secure requests. For this implementation we will manually create a single local CA certificate that will be used to generate a server side certificate and key pair. These certificates and keys will be stored locally for the server.

The server will enforce TLS version 1.3 for maximum security. The server will prefer the TLS13-AES-256-GCM-SHA384 cipher suite to have the best possible encryption standard while maintaining efficient performance. 

### Scoping Requests
In order to scope requests to each client we will create a user for each client, and that will be authenticated against the user in the certificate sent from the client.

#### Future Improvements
* Use a certificate provider for the CA instead of a locally generating one
* Utilize client certificate details to implement more fine grained access
* Implement a dedicated secret management solution for private keys, such as Vault or AWS Secret Manager (or any other secure key store)
* Automate certificate management going forward to remove human error and allow for easier revocation and rotation
* Add monitoring for certificate expiration and authentication failures

### Library
The library will primarily utilize the builtin os package in Go to interact with the Linux operating system and accomplish the actual job management functions.

The library will primarily utilize the os/exec builtin, with some use of the os/user and base level os builtin packages.

### Output streaming
In order to reduce overhead each command will pipe output to a file named with the pid of the process. This data will be read from the file when requested. We will check to see if the process is still running to determine whether or not to continually read from and stream data from the file, or serve all the output data at once to the client. 

In the cases where processes are still running we will utilize server side streaming from gRPC to serve this data continually to the client until they close the connection or the process finishes.

#### Future Improvements
* Store output somewhere other than locally on the server, like S3
* Look into using an external lib that enables tailing from an external source
* Utilize job management to allow suspending and resuming jobs instead of only being able to start and terminate them

## CLI
The CLI will utilize the [cobra](https://github.com/spf13/cobra) CLI library.

### Run a process
The basic structure of the run command will be `run <binary_name> [<args>]`

An example of running a process: `cmdctl run ls -la`

### Get process info
The basic structure of the get-status and get-output retrieval commands will be: `get-status/get-output [<pid>]`

An example of querying for process status: `cmdctl get-status 47854`

We will also have commands for stopping a process and getting the process output. They will follow a similar structure as the above examples.


### Client Authentication
The service will utilize mTLS to authenticate and secure requests. For this implementation we will manually create a single local CA certificate that will be used to generate a client side certificate and key pair. The CA will be trusted by the server to allow client certificates to connect. These certificates and keys will be stored locally for the client.

The client will enforce TLS version 1.3 for maximum security. Each client will need a certificate and key pair generated to interact with the server.

When generating client certificates we will include the the user's identity and the principals on the host they have access to. These will be used as an additional layer of authentication server side to scope access to their user. 

Processes will be run as the user and log output will be stored under the user's home directory.

### Future Improvements
* Create configuration options to allow more robust client side settings
* Auto generate client certificates from the server instead of relying on users to generate their own
* Clients should use the user specified in their cert instead of passing one in
