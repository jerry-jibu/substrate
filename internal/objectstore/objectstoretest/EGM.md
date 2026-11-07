# objectstoretest

## Overview

Test support package providing an in-memory `objectstore.Store` fake, so tests
can observe which external snapshots a flow created and released. This is a
regular (non `_test`) package because callers such as a fake atelet run outside
a test body.

## Interface

- `Fake` (`objectstoretest.go:Fake`): in-memory object store, safe for concurrent use; implements `objectstore.Store` (`objectstoretest.go:Fake` interface assertion).
- `New() *Fake` (`objectstoretest.go:New`): returns an empty Fake.
- `Fake.OnList` / `Fake.OnDelete` / `Fake.OnCopy` (`objectstoretest.go:Fake` fields): hooks that run before each operation and fail it on error; set before handing the Fake to code under test.
- `(*Fake).Put(bucket, object, content string)` (`objectstoretest.go:Put`): writes one object, creating its bucket implicitly.
- `(*Fake).PutSnapshot(t, uri resources.SnapshotURI, names ...string)` (`objectstoretest.go:PutSnapshot`): writes named objects into the snapshot at `uri`.
- `(*Fake).WriteSnapshot(snapshotURI string, names ...string) error` (`objectstoretest.go:WriteSnapshot`): PutSnapshot for callers without a `*testing.T`.
- `(*Fake).Snapshot(t, uri) []string` (`objectstoretest.go:Snapshot`): object names relative to the snapshot prefix.
- `(*Fake).Prefix(t, storagePrefix resources.StoragePrefix) []string` (`objectstoretest.go:Prefix`): object names relative to any storage prefix.
- `(*Fake).Objects() []string` (`objectstoretest.go:Objects`): every object as `bucket/object`, sorted.

## Invariants

- `List` returns names in lexicographic order, matching a real backend, so call-sequence assertions stay meaningful (`objectstoretest.go:List`).
- `Copy` of a missing source fails (`objectstoretest.go:Copy`); buckets are created implicitly by `Put` (`objectstoretest.go:Put`).
- Hook functions run before the operation and their error fails it (`objectstoretest.go:OnList` doc).

## Side Effects

- None. Pure in-memory state guarded by a mutex.

## Verification

- Run: `go test ./internal/objectstore/objectstoretest/...`
