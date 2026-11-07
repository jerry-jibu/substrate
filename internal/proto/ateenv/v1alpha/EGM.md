# v1alpha

## Overview
Generated protobuf and gRPC code for the guest environment API (`guest.proto`, package `ateenv.v1alpha`): the services a guest agent inside a workload uses to run processes and move files, exercised from inside actor sandboxes.

## Interface
- `ateenv.v1alpha.ProcessService` methods (`guest_grpc.pb.go` full-method constants): StartProcess, GetProcess, StreamProcessOutput (server stream), WriteProcessInput (client stream), SignalProcess.
- `ateenv.v1alpha.FileSystemService` methods (`guest_grpc.pb.go:341`): ReadFile (server stream), WriteFile (client stream).
- Client and server stubs: `guest_grpc.pb.go` (`NewProcessServiceClient`, `ProcessServiceServer`, `NewFileSystemServiceClient`, `FileSystemServiceServer`); message types in `guest.pb.go`.

## Invariants
- Regenerated code only: edit `guest.proto` and regenerate, never the `.go` files.

## Side Effects
- ["None"] (generated message and service definitions only).

## Verification
- Run: `go test ./internal/proto/ateenv/v1alpha/...` (generated-code build check).
