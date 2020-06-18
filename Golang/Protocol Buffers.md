# Protocol Buffers

REST is a very common and widely used for end-user-facing APIs, but with the rise of microservices and the amount of communication between them network communication can be a bottleneck.
The network bottleneck has become enough of an issue that companies that operate on a large scale, such as Google and Facebook, have innovated new technologies to speed up communication between microservices.

## Prerequisite

1. Install the compiler: https://developers.google.com/protocol-buffers/docs/downloads.html.

2. Install the Go protocol buffer plugin:

   ```bash
   go get -u github.com/golang/protobuf/protoc-gen-go
   ```

## Introduction to protocol buffers

JSON and XML are commonly used to serialize data.

```txt
Definition:
Serialization is the process of translating data structures or object state into a format that can be stored (for example, in a file or memory buffer) or transmitted (for example, across a network connection link) and reconstructed later.
```

These formats are fairly easy to use and the transfer format is easy to read, but they’re not optimized for transport over a network or for serialization.
Protocol buffers, by Google, are a popular choice for a high-speed transfer format. The data in the messages being transferred over the network is smaller than XML or JSON and it can be marshaled and unmarshaled faster than XML and JSON as well. The transfer method isn’t defined by protocol buffers. They can be transferred on a filesystem, using RPC, over HTTP, via a message queue, and numerous other ways.

A protocol format is defined in a file. It contains the structure of the message and can be used to automatically generate the needed code. 
For example:
user.proto.

```go
package project

message User {
    required string name = 1;
    required int32 id = 2;
    optional string email = 3;
}
```

- The '=1' is a unique tag in the binary encoding.
- We need to specify whether a field is required or optional.
- Because the protocol buffer is used to generate Go code, it’s recommended that it have its own package. In this Go package, the protocol buffer file and generated code can reside.

For more information on protocol bafflers definition we can look at the [DOCs](https://developers.google.com/protocol-buffers/docs/overview).

To compile the protocol buffer to code, we need to generate the code (from the same directory as the .proto file):

```bash
protoc -I=. --go_out=. ./user.proto
# -I specifies the input source directory.
# --go_out indicates where the generated Go source files will go.
# ./user.proto is the name of the file to generate the source from.
```

After the generated code has been created, it can be used to pass messages between a client and server.

For example:
Server:

```go
import (
    "net/http"
    pb "example/userpb"
    "github.com/golang/protobuf/proto"
)

func main() {
    http.HandleFunc("/", func(res http.ResponseWriter, req *http.Request) {
        ....
        body, err := proto.Marshal(user)
        ...
        res.Write(body)
    })
    http.ListenAndServe(":8080", nil)
}
```

Client:

```go
res. err:= http.Get("http://localhost:8080")
if err != nil {
    os.Exit(1)
}
defer res.Body.Close()

b, err := io.ReadAll(res.Body)
if err != nil {
    os.Exit(1)
}
var user pb.User
err = proto.Unmarshal(b, &user)
if err != nil {
    os.Exit(1)
}
....
```

For more information and examples go to the [gotutorial](https://developers.google.com/protocol-buffers/docs/gotutorial)

## Communicating over RPC with protocol buffers

REST has certain semantics it enforces, which include a path and an HTTP verb, and are resource-based. At times, the semantics of REST aren’t desired.

gRPC (www.grpc.io) is an open source, high-performance RPC framework that can use HTTP/2 as a transport layer. It was developed at Google and has support many languages.
gRPC uses code generation to create the messages and handle parts of the communication. This enables the communication to easily work between multiple languages, as the interfaces and messages are generated properly for each language. To define the messages and RPC calls, gRPC uses protocol buffers

```go
syntax = "proto3"

package example

service Hello {
    rpc Say (HelloRequest) return (HelloResponse) {}
}

message HelloRequest {
    string name = 1;
}

message HelloResponse {
    string message= 1;
}
```

For more details on version 3 of protocol buffers, see [Docs](https://developers.google.com/protocol-buffers/docs/proto3).

To generate the code we need to add --go_out=plugins=grpc:. to the protoc command:

```bash
protoc -I=. --go_out=plugins=grpc:. ./hello.proto
```

For example:
Server:

```go
import (
    "net"
    "context"

    pb "example/hellopb"
    "google.golang.org/grpc"
)

type server struct{}

func (s *server) Say(ctx context.Context, in pb.HelloRequest) (*pb.HelloResponse, error) {
    m := "Hello" + in.Name
    return &pb.HelloResponse(Message: msg), nil
}

func main() {
    l, err := net.Listen("txp", ":55555")
    s := grpc.NewServer()
    pb.RegisterHelloServer(s, &server)
    s.Serve(1)
}
```

When the client and server are using HTTP/2, as they are by default when both ends are using gRPC, the communications use the connection reuse and multiplexing. This enables a fast, modern transport layer for transferring the protocol buffer binary messages.

The advantages and disadvantages of RPC utilizing gRPC should be weighed before using them.

The advantages:

- Using protocol buffers, the payload size is smaller and faster to marshal or unmarshal than JSON or XML.
- The context allows for canceling, communicating timeouts or due dates, and other relevant information over the course of a remote call.
- The interaction is with a called procedure rather than a message that’s passed elsewhere.
- Semantics of the transport methodology—for example, HTTP verbs—don’t limit the communications over RPC.

The disadvantages:

- The transport payload isn’t human-readable as JSON or XML can be.
- Applications need to know the interface and details of the of the RPC calls. Knowing the semantics of the message isn’t enough.
- Integration is deeper than a message, such as you’d have with REST, because a remote procedure is being called. Exposing remote procedure access to untrusted clients may not be ideal and should be handled with care for security.

In general, RPCs can be a good alternative for interactions between microservices you control that are part of a larger service. They can provide fast, efficient communication and will even work across multiple programming languages. RPCs shouldn’t typically be exposed to clients outside your control, such as a public API.

## Sources:

- Go in Practice, Published by Manning Publications, 2016.
- 