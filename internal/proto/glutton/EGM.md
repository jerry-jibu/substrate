# glutton

## Overview
Generated protobuf and gRPC code for the glutton load/chaos workload API (`glutton.proto`), used by the benchmarking tools to drive resource-consuming test workloads.

## Interface
- `glutton.Glutton` service methods (`glutton_grpc.pb.go:MethodName`): WriteRAM, ReadRAM, WriteDisk, ReadDisk, OpenFD, Ping, Gossip, BurnCPU, Ingest, UseCPU, RunScript.
- Client and server stubs: `glutton_grpc.pb.go` (`NewGluttonClient`, `GluttonServer`); message types in `glutton.pb.go`.

## Invariants
- Regenerated code only: edit `glutton.proto` and regenerate, never the `.go` files.

## Side Effects
- ["None"] (generated message and service definitions only).

## Verification
- Run: `go test ./internal/proto/glutton/...` (generated-code build check).
