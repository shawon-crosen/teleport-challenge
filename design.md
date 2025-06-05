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
The API will use gRPC to route requests from the client to the server. The API will run locally on the machine that is being managed from the client.

The API will accept requests for running processes as well as requests to return the status and output of a job. More detailed implementation details for the API can be found in the .proto document.

At a high level the API will have the following methods on the localhost using port 9090:

### Run/Stop Methods
`RunProcess(RunProcessRequest)`
`StopProcess(StopProcessRequest)`

### Get Information Methods
`GetProcessStatus(ProcessStatusRequest)`
`GetProcessOutput(ProcessOutputRequest)`

The `RunProcess` method will return a process identifier for the command issued to the client, which will be generated using the username, command requested, and a random string. This identifier will then be used for the `StopProcess`, `GetProcessStatus`, and `GetProcessOutput` methods.

### Future Improvements
* Run the server in a separate environment, such as kubernetes, which can allow for defining high availability through replicaSets and would move the API out from the domain of the operating system being managed
* Utilize some kind of data store (relational or not) to track processes instead of relying on files
* Include resource management of some kind to keep clients from over-taxing the operating system commands are running on
* Scope processes to users that map back to the clients on the operating system

## Server Authentication

### mTLS
The service will utilize mTLS to authenticate and secure requests. For this implementation we will manually create a two local CA certificates, one for host certificates and one for client certificates.

The host CA certificate will be used to generate a server side certificate and key pair.

The client CA certificate will use the `nameConstraints` extension to allow access only to a specific subdomain. In the case of this challenge it will be `permitted:.example.com`.

The server and client will enforce TLS version 1.3 for maximum security. The server will prefer the TLS13-AES-256-GCM-SHA384 cipher suite to have the best possible encryption standard while maintaining efficient performance. 

When generating client certificates we will include the the user's email, in the case of this challenge `shawon@example.com`, in the `subjectAltName` field. This will be used as an additional layer of authentication by the CA to ensure only users with an email matching the allowed subdomain can be validated. Each client will need a client certificate for their user.

### Scoping Requests
In order to scope requests to each client we will create a user for each client, and that will be authenticated against the user in the certificate sent from the client. The user on the host will match the first part of the email, before the `@`, which will be compared against the email we receive from the client in their certificate in the `subjectAltName` field. 

#### Future Improvements
* Use a certificate provider for the CA instead of a locally generating one
* Utilize client certificate details to implement more fine grained access
* Implement a dedicated secret management solution for private keys, such as Vault or AWS Secret Manager (or any other secure key store)
* Automate certificate management going forward to remove human error and allow for easier revocation and rotation
* Add monitoring for certificate expiration and authentication failures

### Library
The library will primarily utilize the standard library's os package in Go to interact with the Linux operating system and accomplish the actual job management functions.

The library will primarily utilize the standard library's os/exec package, with some use of the os/user and base level os library packages.

### Output streaming
In order to reduce overhead each command will pipe output to a file named with the a unique identifier tied to the process. This data will be read from the file when requested. We will check to see if the process is still running to determine whether or not to continually read from and stream data from the file, or serve all the output data at once to the client. 

The flow of this will be something like:
* Read from the file starting at the first line and stream each line to the client
* If at any point we reach the end and get an EOF error, check to see if the process is still running
* * If the process is no longer running, return. 
* If the process is still running, continue reading and streaming from the file while ignoring EOF errors (but break on any other error)

We will utilize server side streaming from gRPC to serve this data continually to the client until they close the connection or the process finishes.

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
The basic structure of the get-status and get-output retrieval commands will be: `get-status/get-output [<unique_process_id>]`

An example of querying for process status: `cmdctl get-status shawon-ls-idhhyqm`

We will also have commands for stopping a process and getting the process output. They will follow a similar structure as the above examples.

We will generate a unique identifier for each process, using the username and a generated value, and write the pid to a file named with that id. This way we can track client processes without collisions, while still having something that ties back to the process itself on the host.

### Client Authentication
Once the user has been validated we will then check to make sure they are authorized to use the gRPC methods defined above. Using the value in `subAltName` from the client certificate, users in the domain `example.com` will be allowed to run the methods we have defined which will be scoped to the following on the host:

* Execute binaries in `/usr/bin` and `/usr/local/bin`
* Read/Write process output files from the output directory

### Future Improvements
* Implement a more robust form of authorization, using something like JWT or Oauth 2.0 to restrict access to methods
* Create configuration options to allow more robust client side settings
* Auto generate client certificates from the server instead of relying on users to generate their own
