# ateletpb

## Overview
Generated protobuf and gRPC code for the atelet API (`atelet.proto`): the `atelet.Atelet` service ate-api-server calls on an atelet, and the `atelet.WorkerSupport` service an atelet calls back into ate-api-server for. Consumers include `cmd/ateapi` and `cmd/atelet`.

## Interface
- `atelet.Atelet` service methods (`atelet_grpc.pb.go:MethodName`): Run, Checkpoint, Restore, PromoteSnapshot, DetachActor, Terminate.
- `atelet.WorkerSupport` service methods (`atelet_grpc.pb.go:MethodName`): MintActorCertificate, AcquireWorkerEpoch, RegisterWorker, SetWorkerSandboxRuntimes, RequestActorSuspend, GetActorEgressPolicy, ReportActorCrashed.
- Client and server stubs: `atelet_grpc.pb.go` (`NewAteletClient`, `AteletServer`, `NewWorkerSupportClient`, `WorkerSupportServer`); message types in `atelet.pb.go`.

## Invariants
- Regenerated code only: do not edit `atelet.pb.go` or `atelet_grpc.pb.go` by hand; change `atelet.proto` and regenerate.
- Per AGENTS.md, atelet and the worker runtimes share only `internal/proto/*` types; this package is part of that allowed surface.

## Side Effects
- ["None"] (generated message and service definitions only).

## Verification
- Run: `go test ./internal/proto/ateletpb/...` (generated-code build check).
