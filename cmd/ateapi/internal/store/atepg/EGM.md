# atepg

## Overview
The PostgreSQL implementation of `store.Interface` ("Package atepg", `atepg.go`). Each table keeps native SQL columns for the fields SQL must operate on (keys, versions, pagination, preconditions) plus the full protobuf message, binary-encoded, in a BYTEA column. TLS comes entirely from the libpq connection string.

## Interface
- `atepg.go:Persistence` implements `store.Interface` (compile-checked by `var _ store.Interface = (*Persistence)(nil)`).
- `atepg.go:Connect` opens the read/write, owner, watch, and probe pools, creates the schema, and applies migrations; `atepg.go:ConnectConfig` carries the DSNs, roles, schema, and pool size. `atepg.go:NewPersistence` wraps an existing `pgxpool.Pool`. `atepg.go:ErrUnavailable` reports a failed initial connection.
- Resource CRUD: `actor.go:Persistence.CreateActor`/`UpdateActor`/`MarkActorForDeletion`/`DeleteActor`/`ListActors`/`GetActor`, `worker.go:CreateWorker` and siblings, `tag.go:CreateTag` and siblings, `actor_template.go:CreateActorTemplate` and siblings, `atespace.go:CreateAtespace` and siblings, `egress_policy.go:CreateEgressPolicy` and siblings, `access_policy.go:CreateGlobalAccessPolicy` and `CreateAtespaceAccessPolicy` and siblings.
- Worker assignment: `worker_assignment.go:Persistence.BindActorToWorker`, `UpdateActorAndRelease`, `GetWorkerAssignment`, `ListWorkerAssignments`.
- Cross-instance coordination: `lease.go:Persistence.AcquireLease`, `outbox.go:Persistence.WatchWorkers`.
- Lifecycle: `atepg.go:Persistence.Ping`, `Close`, `Pool`, `SetPolicyManager`.
- Child: `./migrations/EGM.md` (Goose SQL migrations).

## Invariants
- Pool separation: `atepg.go:Persistence` docs fix `watchPoolMaxConns = 3`, `ownerPoolMaxConns = 2`, `probePoolMaxConns = 1`, so worker-event polling, migrations/outbox maintenance, and the readiness probe cannot starve each other.
- Worker events reach watchers only after the writing transaction commits, and delivery is fenced by transaction order (`outbox.go:WatchWorkers`, `atepg.go:publishLocally`).
- Local publish is best effort: `atepg.go:publishLocally` skips watchers with full buffers; they catch up from the outbox poll.
- Outbox partitions are unlogged and created with `autovacuum_enabled = off` (`outbox.go` partition DDL).
- Updates and deletes are CAS-guarded on uid and version, matching `store.go` preconditions.
- `Connect` refuses to run without an explicit non-empty schema and configured roles (`atepg.go:Connect` guards).

## Side Effects
- Reads and writes PostgreSQL: all resource tables plus the `worker_outbox` partitioned table and lease rows.
- Runs background loops: expired-lease cleanup and outbox partition maintenance (`atepg.go:Persistence.stopMaintenance`).
- Serves in-process worker events to `WatchWorkers` channels (`atepg.go:addWatcher`).
- Emits OTel spans for queries (`tracing.go` query tracer).

## Verification
- Run: `go test ./cmd/ateapi/internal/store/atepg/...` (requires Docker for the PostgreSQL tests; see `dockerenv`).
