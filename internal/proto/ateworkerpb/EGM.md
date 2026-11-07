# ateworkerpb

## Overview
Generated protobuf and gRPC code for the ateworker API (`ateworker.proto`): the `ateworker.Worker` service the atelet calls on the worker runtime (gVisor or microVM) to run, checkpoint, restore, terminate, and observe workloads. Consumers include `cmd/ateworker-gvisor` and `cmd/ateworker-microvm`.

## Interface
- `ateworker.Worker` service methods (`ateworker_grpc.pb.go:MethodName`): RunWorkload, CheckpointWorkload, RestoreWorkload, GetWorkloadStats, GetActiveWorkloadStats, TerminateWorkload.
- Client and server stubs: `ateworker_grpc.pb.go` (`NewWorkerClient`, `WorkerServer`); message types in `ateworker.pb.go`, including `RunWorkloadRequest`, `ActorDirs`, `WorkloadSpec`, `Container`, and `SnapshotFidelity`.

## Invariants
- Regenerated code only: edit `ateworker.proto` and regenerate, never the `.go` files.
- Version skew policy applies: this proto is shared by atelet and workers across versions (see `docs/reference/version-skew-policy.md` per AGENTS.md).

## Side Effects
- ["None"] (generated message and service definitions only).

## Verification
- Run: `go test ./internal/proto/ateworkerpb/...` (generated-code build check).
