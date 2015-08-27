# grpc-gateway

[![Build Status](https://travis-ci.org/gengo/grpc-gateway.svg?branch=master)](https://travis-ci.org/gengo/grpc-gateway)

grpc-gateway is a plugin of [protoc](http://github.com/google/protobuf).
It reads [gRPC](http://github.com/grpc/grpc-common) service definition,
and generates a reverse-proxy server which translates a RESTful JSON API into gRPC.

It helps you to provide your APIs in both gRPC and RESTful style at the same time.

## Background
gRPC is great -- it generates API clients and server stubs in many programming languages,
it is fast, easy-to-use, bandwidth-efficient and its design is combat-proven by Google.
However, you might still want to provide classic RESTful APIs too for some reasons --
compatibility with languages not supported by gRPC, API backward-compatibility or aesthetics
of RESTful architecture.

That's what grpc-gateway helps you to do. You just need to implement your gRPC service with a small amount of custom options.
Then the reverse-proxy generated by grpc-gateway serves RESTful API on top of the gRPC service.

## Installation
First you need to install ProtocolBuffers 3.0 or later.

```sh
mkdir tmp
cd tmp
git clone https://github.com/google/protobuf
./autogen.sh
./configure
make
make check
sudo make install
```

Then, `go get` as usual.

```sh
go get github.com/gengo/grpc-gateway/protoc-gen-grpc-gateway
go get github.com/golang/protobuf/protoc-gen-go
```
 
## Usage
Make sure that your `$GOPATH/bin` is in your `$PATH`.

1. Define your service in gRPC
   
   your_service.proto:
   ```protobuf
   syntax = "proto3";
   package example;
   message StringMessage {
     string value = 1;
   }
   
   service YourService {
     rpc Echo(StringMessage) returns (StringMessage) {}
   }
   ```
2. Add a custom option to the .proto file
   
   your_service.proto:
   ```diff
    syntax = "proto3";
    package example;
   +
   +import "google/api/annotations.proto";
   +
    message StringMessage {
      string value = 1;
    }
    
    service YourService {
   -  rpc Echo(StringMessage) returns (StringMessage) {}
   +  rpc Echo(StringMessage) returns (StringMessage) {
   +    option (google.api.http) = {
   +      post: "/v1/example/echo"
   +      body: "*"
   +    };
   +  }
    }
   ```
3. Generate gRPC stub
   
   ```sh
   protoc -I/usr/local/include -I. \
     -I$GOPATH/src \
     -I$GOPATH/src/github.com/gengo/grpc-gateway/third_party/googleapis \
     --go_out=plugins=grpc:. \
     path/to/your_service.proto
   ```
   
   It will generate a stub file `path/to/your_service.pb.go`.
4. Implement your service in gRPC as usual
   1. (Optional) Generate gRPC stub in the language you want.
     
     e.g.
     ```sh
     protoc -I/usr/local/include -I. \
       -I$GOPATH/src \
       -I$GOPATH/src/github.com/gengo/grpc-gateway/third_party/googleapis \
       --ruby_out=. \
       path/to/your/service_proto
     
     protoc -I/usr/local/include -I. \
       -I$GOPATH/src \
       -I$GOPATH/src/github.com/gengo/grpc-gateway/third_party/googleapis \
       --plugin=protoc-gen-grpc-ruby=grpc_ruby_plugin \
       --grpc-ruby_out=. \
       path/to/your/service.proto
     ```
   2. Implement your service
5. Generate reverse-proxy
   
   ```sh
   protoc -I/usr/local/include -I. \
     -I$GOPATH/src \
     -I$GOPATH/src/github.com/gengo/grpc-gateway/third_party/googleapis \
     --grpc-gateway_out=logtostderr=true:. \
     path/to/your_service.proto
   ```
   
   It will generate a reverse proxy `path/to/your_service.pb.gw.go`.
6. Write an entrypoint
   
   Now you need to write an entrypoint of the proxy server.
   ```go
   package main
   import (
     "flag"
     "net/http"
   
     "github.com/golang/glog"
     "golang.org/x/net/context"
     "github.com/gengo/grpc-gateway/runtime"
   	
     gw "path/to/your_service_package"
   )
   
   var (
     echoEndpoint = flag.String("echo_endpoint", "localhost:9090", "endpoint of YourService")
   )
   
   func run() error {
     ctx := context.Background()
     ctx, cancel := context.WithCancel(ctx)
     defer cancel()
   
     mux := runtime.NewServeMux()
     err := gw.RegisterYourServiceHandlerFromEndpoint(ctx, mux, *echoEndpoint)
     if err != nil {
       return err
     }
   
     http.ListenAndServe(":8080", mux)
     return nil
   }
   
   func main() {
     flag.Parse()
     defer glog.Flush()
   
     if err := run(); err != nil {
       glog.Fatal(err)
     }
   }
   ```

## More Examples
More examples are available under `examples` directory.
* `examplepb/echo_service.proto`, `examplepb/a_bit_of_everything.proto`: service definition
  * `examplepb/echo_service.pb.go`, `examplepb/a_bit_of_everything.pb.go`: [generated] stub of the service
  * `examplepb/echo_service.pb.gw.go`, `examplepb/a_bit_of_everything.pb.gw.go`: [generated] reverse proxy for the service
* `server/main.go`: service implementation
* `main.go`: entrypoint of the generated reverse proxy

## Features
### Supported
* Generating JSON API handlers
* Method parameters in request body
* Method parameters in request path
* Method parameters in query string
* Mapping streaming APIs to JSON streams
* Mapping HTTP headers with `Grpc-Metadata-` prefix to gRPC metadata

### Want to support
But not yet.
* bytes and enum fields in path parameter. #5
* Optionally generating the entrypoint. #8
* Optionally emitting API definition for [Swagger](http://swagger.io). #9

### No plan to support
But patch is welcome.
* Method parameters in HTTP headers
* Handling trailer metadata
* Encoding request/response body in XML
* True bi-directional streaming. (Probably impossible?)

# License
grpc-gateway is licensed under the BSD 3-Clause License.
See `LICENSE.txt` for more details.
