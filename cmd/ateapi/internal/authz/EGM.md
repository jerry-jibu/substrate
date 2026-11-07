# ateapi/internal/authz

## Overview

Authorization for the ateapi control plane: an embedded OpenFGA server backed by
the same PostgreSQL pool as the resource store, a read-path `Authorizer`, a
write-path `PolicyManager`, per-RPC gRPC interceptors driven by
`defaultRPCPermissions`, and object/user string helpers.

## Interface

- `New`, `NewOpenFGAServer`, `EnsureStoreAndModel`, `SystemPrincipals` (`authz_util.go`) - provision the default OpenFGA store and the checked-in authorization model, and return the read-path and write-path components; bootstrap owners are granted global ownership.
- `AtespaceObject`, `WorkerObject`, `ActorObject`, `ActorTemplateObject`, `TagObject`, `FormatMember`, `GlobalRootObject` (`authz_util.go`, `registry.go`) - OpenFGA object and member string formatting.
- `WithBypass`, `IsBypassed` (`authz_util.go`) - contexts that bypass runtime authorization checks, for internal system reconcilers.
- `Authorizer` / `Check` (`authorizer.go`) - the read-only policy decision point; structural hierarchy links (global:root as parent of every atespace) are supplied as ContextualTuples at evaluation time rather than persisted.
- `PolicyManager` / `ReconcileGlobalBindings`, `ReconcileAtespaceBindings`, `DeleteAtespacePolicies` (`policy_manager.go`) - OpenFGA tuple writes for access policy and atespace bindings.
- `UnaryServerInterceptor`, `StreamServerInterceptor` (`interceptor.go`) - per-RPC enforcement from `defaultRPCPermissions` (`registry.go`); the stream interceptor denies every streaming RPC.
- `ContextWithTx`, `TxFromContext` (`tx_context.go`), `ErrNoTransactionInContext` (`datastore.go`) - pgx transaction injection for the datastore.
- Transaction-aware PostgreSQL datastore over OpenFGA migration version 6 (`datastore.go`: `ReadAuthorizationModel`, `ReadPage`, `Write`, `Close`).

## Invariants

- Every served unary RPC needs an entry in `defaultRPCPermissions`; an RPC without one is denied and a test fails the build (`registry.go`; see `TestDefaultRPCPermissions_CoversEveryServedRPC`). gRPC reflection is open to authenticated callers.
- A check is never skipped: every resource identifier a check needs must be present and a valid resource name, otherwise the extractor returns InvalidArgument; unexpected request types are Internal and the interceptor fails closed (`registry.go` `checkExtractor`).
- The datastore enforces that `ReadPage` and `Write` run inside an active pgx.Tx supplied via `ContextWithTx` (`datastore.go`, `ErrNoTransactionInContext`).
- `PolicyManager` tuple writes without a PostgreSQL transaction fail with `ErrNilTransaction` (`policy_manager.go`).
- Resource mutations and authorization tuple updates execute in the same transaction, which is why the OpenFGA schema lives in the ateapi schema (`datastore.go`; `../atepg/migrations`).
- OpenFGA object IDs may not contain ':', '#', or whitespace, and '*' is a public wildcard, so `FormatMember` validates member strings before they reach OpenFGA (`authz_util.go`).

## Side Effects

- Runs OpenFGA background workers (`NewOpenFGAServer`); `Close` stops them without closing the caller-owned pool.
- Reads and writes OpenFGA tables (`tuple`, `changelog`, `store`, ...) in the shared PostgreSQL schema.

## Verification

- Run: `go test ./cmd/ateapi/internal/authz/...`
