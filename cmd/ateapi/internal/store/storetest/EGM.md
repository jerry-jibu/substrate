# ateapi/internal/store/storetest

## Overview

Provides isolated PostgreSQL-backed stores for tests. One PostgreSQL container
is shared by every test in a package; each test gets its own database
(`storetest.go` package doc).

## Interface

- `SetupTestStore` (`storetest.go`) - a `store.Interface` over a fresh per-test database.
- `SetupPostgresPersistence` (`storetest.go`) - an isolated `*atepg.Persistence`, with an OpenFGA server and PolicyManager wired the way the server wires them.
- `DefaultSystemPrincipals` (`storetest.go`) - valid `authz.SystemPrincipals` for the default install namespace.
- Fixtures: `MustCreateAtespace`, `MustCreateActor`, `MustCreateTag`, `MustSeedAssignment`, `WorkerHostingActor` (`storetest.go`).
- `RunTests`, `Shutdown` (`storetest.go`) - TestMain driver that terminates the shared container; packages with other TestMain work must call `Shutdown` themselves.

## Invariants

- Each test's database is created and dropped around it, so subtests can run in parallel against the one container (`storetest.go` `SetupPostgresPersistence`).
- `MustSeedAssignment` stamps metadata the way `BindActorToWorker` does, but skips the Actor update it commits with (`storetest.go`).
- When the container is unavailable, the test fails under `dockerenv.Required()` and skips otherwise (`storetest.go` `requireAdminPool`).

## Side Effects

- Starts a `postgres:18-alpine` testcontainer, creates and drops databases, and executes raw SQL against the `worker_assignments` table (`storetest.go`).

## Verification

- Run: `go test ./cmd/ateapi/internal/store/storetest/...`
