# grpcechopb

## Overview
Generated protobuf and gRPC code for the gRPC echo test service (`grpcecho.proto`), used by e2e and benchmarking tests to check end-to-end gRPC connectivity.

## Interface
- `grpcecho.Echo` service methods (`grpcecho_grpc.pb.go` `Echo_*_FullMethodName` consts): Echo (unary), EchoStream (server stream), EchoBidi (bidi stream).
- Client and server stubs: `grpcecho_grpc.pb.go` (`NewEchoClient`, `EchoServer`); message types in `grpcecho.pb.go`.

## Invariants
- Regenerated code only: edit `grpcecho.proto` and regenerate, never the `.go` files.

## Side Effects
- ["None"] (generated message and service definitions only).

## Verification
- Run: `go test ./internal/proto/grpcechopb/...` (generated-code build check).
