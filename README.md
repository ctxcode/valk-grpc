# valk-grpc

Pure-[Valk](https://valk-lang.dev) gRPC server and client over HTTP/2 (`h2c` cleartext and TLS ALPN `h2`) with full streaming, metadata, trailers, deadlines, gzip compression, and `valk-protobuf` integration.

Requires Valk 0.9.2 or newer.

## Install

```
vman install github.com/zikue/valk-grpc
```

## Features

- **Full gRPC over HTTP/2 (`h2c` & TLS `h2`)**:
  - Multiplexed concurrent streams over a single HTTP/2 connection with RFC 9113 framing, flow control (`WINDOW_UPDATE`), `PING`/`PONG`, `RST_STREAM`, `GOAWAY`, and RFC 7541 HPACK (including Huffman decoding).
  - Supports cleartext HTTP/2 (`h2c` prior knowledge) and TLS with HTTP/2 ALPN (`h2`) including optional mutual TLS.
- **All 4 RPC modes**:
  - Unary, server-streaming, client-streaming, and full-duplex bidirectional streaming on both client and server.
- **Protocol conformance**:
  - Standard gRPC `StatusCode` (`0`–`16`) and tagged `GrpcError` union (with `code`, `message`, binary `details`, and `to_string() $auto`).
  - Length-prefixed message framing with optional `gzip` compression (`grpc-encoding` / `grpc-accept-encoding`) and configurable max message size enforcement (`RESOURCE_EXHAUSTED`).
  - Request headers, response headers, response trailers, trailers-only error responses, binary `-bin` base64 metadata, `grpc-message` percent-encoding/decoding, and `grpc-timeout` deadline propagation.
- **Server interceptors & graceful shutdown**:
  - Composable unary and stream interceptors (`add_unary_interceptor`, `add_stream_interceptor`) and graceful `shutdown()` via HTTP/2 `GOAWAY`.
- **Seamless `valk-protobuf` integration**:
  - Generic protobuf helpers (`call_unary[Req, Res]`, `TypedClientStream[Req, Res]`, `TypedServerStream[Req, Res]`) plus generated service interfaces, registrars, and client stubs via `valk-protoc --grpc`.

## Usage

### Server & Client (Raw Bytes)

```valk
use grpc

fn main() {
    let srv = grpc.GrpcServer.new("127.0.0.1", 50051)

    srv.register_unary("/demo.Echo/Say", fn(ctx: grpc.CallContext, req: String) String !grpc.GrpcError {
        ctx.set_header("x-server", "valk-grpc")
        ctx.set_trailer("x-done", "1")
        return "echo: " + req
    })

    srv.register_bidi_stream("/demo.Echo/Chat", fn(stream: grpc.ServerStream) void !grpc.GrpcError {
        while true {
            let msg = (stream.recv() !>) ?! break
            stream.send("reply: " + msg) !>
        }
    })

    let port = srv.bind() ! panic("Bind failed: %{E}")
    co srv.start()
    defer srv.shutdown(5000)

    let client = grpc.GrpcClient.connect("127.0.0.1:%{port}") ! panic("Connect failed: %{E}")
    defer client.close()

    let resp = client.unary("/demo.Echo/Say", "hello") ! panic("RPC failed: %{E}")
    println(resp.body)
}
```

### With `valk-protobuf` Generated Stubs (`valk-protoc --grpc`)

Generate typed messages, `<Service>Service`, `register_<Service>_service`, and `<Service>Client` from a `.proto` file:

```sh
valk-protoc greeter.proto --grpc -o greeter.valk
```

Implement the generated service interface and call via the generated client:

```valk
use grpc

class MyGreeter is GreeterService {
    + fn say_hello(ctx: grpc.CallContext, req: HelloRequest) HelloReply !grpc.GrpcError {
        let _ = ctx
        return HelloReply { message: "Hello, %{req.name}!" }
    }
}

fn run() {
    let srv = grpc.GrpcServer.new("127.0.0.1", 0)
    register_Greeter_service(srv, MyGreeter {})
    let port = srv.bind() ! panic(E.to_string())
    co srv.start()
    defer srv.shutdown(2000)

    let conn = grpc.GrpcClient.connect("127.0.0.1:%{port}") ! panic(E.to_string())
    defer conn.close()
    let stub = GreeterClient.new(conn)

    let reply = stub.say_hello(HelloRequest { name: "Valk" }) ! panic(E.to_string())
    println(reply.message)
}
```

### TLS (`h2`)

```valk
let srv = grpc.GrpcServer.new("127.0.0.1", 50051)
srv.tls("cert.pem", "key.pem") ! panic(E.to_string())

let client = grpc.GrpcClient.connect("https://127.0.0.1:50051", grpc.ClientOptions {
    tls: true
    ca_cert_path: "ca.pem"
}) ! panic(E.to_string())
```

## Development

- `make lint` checks sources with `--lint --deny-warnings`
- `make test` runs unit, HTTP/2 loopback (`h2c` & TLS `h2`), and `valk-protoc --grpc` end-to-end tests
- `make example` runs the runnable example
- `make docs` regenerates `docs/api.md` and `docs/api-full.md`
