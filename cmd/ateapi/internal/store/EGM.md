# store

## Overview
The persistence layer contract for ate-api-server. `store.go:Interface` defines every storage operation the control plane performs over actors, workers, tags, templates, policies, and atespaces, plus the sentinel errors callers branch on. Implementations live in `atepg` (PostgreSQL); `storetest` and `storecontract` verify implementations against this contract.

## Interface
- `store.go:Interface`: `CreateActor`, `GetActor`, `ListActors`, `UpdateActor`, `MarkActorForDeletion`, `DeleteActor`, egress policy CRUD (`CreateEgressPolicy` and siblings), and the worker/tag/template/atespace/access-policy/lease/watch operations implementers must provide.
- Sentinel errors: `store.go:ErrNotFound`, `ErrAlreadyExists`, `ErrVersionConflict`, `ErrFailedPrecondition`, `ErrWorkerNotFound`, `ErrLeaseConflict`, `ErrInvalidPageToken`, `ErrInvalidPageSize`, `ErrUIDConflict`, `ErrPreconditionRequired`, `ErrImmutableField`.
- Support types: `store.go:Precondition`, `DeletePreconditions`, `ListOptions`, `ListResponse`, `Lease`, `WorkerWatch`, `WorkerEvent`.
- Children: `./atepg/EGM.md` (PostgreSQL implementation), `./storecontract/EGM.md` (contract test suite), `./storetest/EGM.md` (PostgreSQL-container-backed test store), `./dockerenv/EGM.md` (CI Docker check).

## Invariants
- Updates require preconditions carrying both uid and version guards; the store rejects blind writes (`store.go:Interface.UpdateActor` doc, `ErrPreconditionRequired`).
- `mutate` callbacks may run more than once: the store retries a write that loses a concurrent race (`store.go:Interface.UpdateActor` doc).
- `DeleteActor` fails with `ErrFailedPrecondition` while the actor still has `status.worker_assignment` set (`store.go:Interface.DeleteActor` doc).
- Listing supports cross-atespace scans when `atespace` is empty; page tokens are scoped to one list operation (`store.go:Interface.ListActors` doc, `ErrInvalidPageToken`).
- UID conflicts (`ErrUIDConflict`) are not retryable: the name now addresses a different object incarnation (`store.go:ErrUIDConflict` doc).

## Side Effects
- Implementations write the authoritative actor state database (`atepg` uses PostgreSQL).
- `WorkerWatch` streams committed worker events to in-process watchers.

## Verification
- Run: `go test ./cmd/ateapi/internal/store/...`
