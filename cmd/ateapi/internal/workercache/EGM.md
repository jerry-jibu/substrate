# ateapi/internal/workercache

## Overview

Maintains an in-memory view of all workers, kept current via
`store.Interface.WatchWorkers`. It exposes `Workers()` for fast O(1) reads
during actor scheduling (`workercache.go` package doc).

## Interface

- `New(store, relistInterval)` (`workercache.go`) - a Cache over the store's worker list/watch.
- `Start` (`workercache.go`) - syncs synchronously, then streams updates, relists periodically, and resyncs on connection loss; returns once the initial sync succeeds.
- `Workers` (`workercache.go`) - a snapshot of all known workers; the returned slice and elements must not be modified by the caller.
- `AddHandler` (`workercache.go`) - handler invoked for every current Worker, every watch create/update, and every Worker a relist finds new or at a different version.
- `Worker(name)` / `Forget(name)` (`workercache.go`) - point lookup of one worker; `Forget` removes a worker a store write proved no longer exists, closing the select-then-delete race after a failed scheduling write.
- `workerListWatcher` (`workercache.go`) - the exact storage interface needed: `WatchWorkers` and `ListWorkers`.

## Invariants

- `Workers` returns an error while the cache is not ready (brief window during reconnect); callers are expected to retry (`workercache.go` `Workers`).
- Handlers run on the goroutine applying the change and must not block (`workercache.go` `AddHandler`).
- The relist (`relistPageSize = 1000`) recovers from state drift caused by missed WorkerWatch events (`workercache.go` `New`, `relistPageSize`).

## Side Effects

- Runs a background goroutine that streams worker watch events and relists on an interval (`workercache.go` `Start`).

## Verification

- Run: `go test ./cmd/ateapi/internal/workercache/...`
