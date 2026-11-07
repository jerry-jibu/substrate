# filecache

## Overview

Node-local disk cache of downloaded artifacts (sandbox assets, snapshots),
keyed by opaque identities. An artifact wanted by N concurrent callers is
fetched once (singleflight), publication is one atomic rename, and entries
are evicted under byte-budget pressure without breaking consumers. The store
owns its root exclusively (one atelet per node).

## Interface

- `store.go:New` - opens (creating if needed) the store rooted at `root`; options `WithMinAge`, `WithFetchTimeout`.
- `key.go:Key`, `key.go:SHA256Key`, `key.go:URIKey` - cache identities; keys are built via constructors only, the zero Key is invalid.
- `getfileto.go:GetFileTo` - materializes a single-file artifact at `dst` as a hard link to the read-only cache copy, fetching with a `FileFetcher` on miss.
- `getdirto.go:GetDirTo` - materializes a directory-tree artifact at `dst` as a caller-owned tree of hard links, fetching with a `DirFetcher` on miss.
- `getfilecopyto.go:GetFileCopyTo` - materializes a private copy the consumer owns and may mutate.
- `evict.go:EvictUnused` - frees space until `targetBytes` are reclaimed, LRU first; returns `EvictStats` and supports `dryRun`.
- `evict.go:TotalBytes` - sums allocated (on-disk) bytes of all published entries.
- `sweep.go:SweepDebris` - removes crash debris (`tmp/` children and `.rm-*` dirs); returns `SweepStats`.
- On-disk layout: `entries/<sha256(key)>/{data,meta.json}`, `tmp/`, `.rm-*` (`doc.go`).

## Invariants

- Eviction never takes what a caller holds: `GetFileTo` serves hits as hard links created inside `hitMu`, `GetDirTo` builds its linked tree in the same locked step, and a linked entry (Nlink > 1) is never retired (`doc.go`, `evict.go:sizeEntry`, `evict.go:retireEntry`).
- Entries younger than the store's min age (default 10 minutes, `defaultMinAge`) are vetoed for eviction, covering the window between publication and the consumer's use becoming visible (`evict.go:EvictUnused`, `store.go:defaultMinAge`).
- Published files are read-only (`0o444` keeping execute bits, `getfileto.go:readOnlyMode`): a consumer writing through its hard link fails with EACCES instead of corrupting shared bytes; mutating consumers use `GetFileCopyTo`.
- Keys are identities, not addresses; the store has no invalidation, so a key must name content immutable at its source and changed content must arrive under a new key (`doc.go`, `key.go:URIKey`).
- Publication is atomic and crash-safe: fetches land in `tmp/` and one rename moves them under `entries/`; everything under `entries/` is complete (`doc.go`, `getfileto.go:fetchAndPublish`).
- Fetches run on a context detached from callers, bounded only by the store's fetch timeout (default 10 minutes), so one canceled caller never aborts a download others await (`getfileto.go:fetchFlight`, `store.go:defaultFetchTimeout`).
- Serving an entry through a method of the other kind (file vs tree) is an error, not a miss (`getfileto.go:entryKind`, `getfileto.go:linkOut`).
- A destination must be an absolute path that does not exist yet, on the cache's filesystem (`getfileto.go:getTo`, `getfileto.go:linkOut` EXDEV check).
- `SweepDebris` runs once at startup, before the store serves; a later sweep would delete in-flight fetches' working dirs (`sweep.go:SweepDebris`).
- The read-through loop retries at most `linkRetries` (3) times before concluding something else is deleting under the store root (`getfileto.go:linkRetries`).
- `meta.json` is debugging-only; nothing on a correctness path reads it (`doc.go`, `store.go:entryMeta`).

## Side Effects

- Filesystem: creates `entries/` (0755) and `tmp/` (0700) under the store root; writes, hard links, renames, `os.Chtimes` last-use touches, and `os.RemoveAll` of retired entries (`store.go:New`, `evict.go:EvictUnused`).
- Runs caller-supplied fetch functions (`FileFetcher`/`DirFetcher`) in subprocess-free callbacks that themselves may touch the network or disk.
- In-memory: singleflight group, `hitMu` RWMutex, `evictMu` mutex (`store.go:Store`).

## Verification

- Run: `go test ./cmd/atelet/internal/filecache/...`
