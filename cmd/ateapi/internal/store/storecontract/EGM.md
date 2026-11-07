# ateapi/internal/store/storecontract

## Overview

Backend-neutral assertions for `store.Interface` implementations
(`contract.go` package doc). Any store backend runs these through
`RunContractTests`; PostgreSQL-specific behavior such as foreign-key races and
transactional notifications stays in atepg's own tests.

## Interface

- `RunContractTests` (`contract.go`) - runs every contract suite (actors, egress policies, workers, worker assignments, atespaces, actor templates, tags, leases, list options, unknown fields) against a fresh `store.Interface` built by `setup`.
- Fixtures and helpers (`contract.go`): `newTestActorTemplate`, `newTestSuspendedActor`, `newTestInProgressTag`, `newTestWorker`, `mustCreateAtespace`, `assertPruned`, `requireNoUnknownFields`, `testMarkForDeletion`.

## Invariants

The contract the assertions enforce, all in `contract.go`:

- Create assigns uid and version 1, sets create/update time, and does not mutate its input (`CreateActor_Success`).
- Unguarded updates, or guards on only a uid or only a version, fail with `ErrPreconditionRequired`; stale versions fail with `ErrVersionConflict`; foreign uids with `ErrUIDConflict` (`UpdateActor_MissingPrecondition`, `*_Preconditions`).
- A mutation that returns an error writes nothing (`UpdateActor_MutateError`).
- Server-owned metadata (uid, create time) is derived from the stored resource, never accepted from a mutation (`UpdateActor_Success`).
- Only a `Mark*ForDeletion` sets `delete_time`; a plain update cannot set or clear it, and a repeated mark keeps the first stamp (`testMarkForDeletion`).
- Deleting an Actor deletes its egress policy with it, and a replacement Actor does not inherit it (`EgressPolicy_ActorLifecycle`).
- An atespace holding templates or tags cannot be deleted (`ErrFailedPrecondition`) (`ActorTemplateResources_BlockAtespaceDeletion`, `Tag_Lifecycle`).
- A reserved tag's snapshot storage starts PENDING; once recorded, snapshot URI, fidelity, and template uid are immutable (`ErrImmutableField`) (`Tag_Lifecycle`, `Tag_ImmutableFields`).
- Negative `PageSize` fails with `ErrInvalidPageSize` and a malformed `PageToken` with `ErrInvalidPageToken` (`runListOptionsContractTests`).
- Stored results carry no unknown protobuf fields, so unknown client fields are pruned (`runUnknownFieldContractTests`, `assertPruned`).

## Side Effects

- ["None"] (all effects are against the `store.Interface` the caller supplies).

## Verification

- Run: `go test ./cmd/ateapi/internal/store/storecontract/...`
