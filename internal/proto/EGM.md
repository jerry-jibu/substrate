# proto

## Overview
Internal protobuf and gRPC definitions shared between the ate system components. Per AGENTS.md, `internal/proto/*` is the only contract atelet and the worker runtimes may share. Each subdirectory holds the `.proto` source and its generated Go code; `ateenv` is versioned under `v1alpha`.

## Subdirectories & Boundaries
- `ateletpb`: `./ateletpb/EGM.md` - `atelet.Atelet` and `atelet.WorkerSupport` services between ate-api-server and atelets.
- `ateworkerpb`: `./ateworkerpb/EGM.md` - `ateworker.Worker` service between atelets and worker runtimes.
- `objectstoreexchangepb`: `./objectstoreexchangepb/EGM.md` - `objectstoreexchange.CredentialExchange` service for snapshot credential exchange.
- `ateenv`: `./ateenv/EGM.md` - guest environment API namespace (`v1alpha`).
- `glutton`: `./glutton/EGM.md` - load/chaos workload service for benchmarking.
- `grpcechopb`: `./grpcechopb/EGM.md` - gRPC echo test service.
