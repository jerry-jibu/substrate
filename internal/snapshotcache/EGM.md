# snapshotcache

## Overview

Disposes of staged restore files and evicts preserved snapshot files from
the host page cache after an actor restore completes. Linux-only.

## Interface

- `snapshotcache.go:File` - a snapshot data file pinned by `fs.FileInfo` at discovery time, so a delayed pass cannot evict or unlink a newer file written at the same path.
- `snapshotcache.go:FindFiles` - walks a dir for regular files whose base name matches.
- `snapshotcache.go:DropFile`/`DropFiles` - unlink a file only if it still refers to the same inode and size.
- `snapshotcache.go:StartWriteback` - opens paths and, in the background, polls until each file's dirty pages are written back or the file is unlinked (`snapshotcache.go:writebackPollDelays`, 25ms..3s), then calls `done` once; an unopenable path returns the error and never calls `done`.
- `snapshotcache.go:EvictFiles` - `POSIX_FADV_DONTNEED` on clean cached pages; files with dirty or writeback pages are skipped and reported as remaining; cachestat failures are warned once (`snapshotcache.go:cachestatWarnOnce`).
- `snapshotcache.go:EvictWithRetries` - evict plus retries after each `snapshotcache.go:RetryDelays` interval (0.5s/1.5s/3s) while pages remain.
- Test seams `snapshotcache.go:cachestatFn`/`fadviseFn` allow deterministic tests.

## Invariants

- `FADV_DONTNEED` is never issued on a file with dirty or writeback pages, so it never forces synchronous disk writeback (`snapshotcache.go:EvictFiles` doc).
- Writeback polls instead of waiting on write-out, so a file unlinked meanwhile is closed and its dirty pages dropped rather than written back (`snapshotcache.go:startWriteback` comment).
- Write-out is submitted 64 MiB at a time (`snapshotcache.go:writeOutChunk`) and stops once the file is unlinked.
- Without cachestat (pre-Linux 6.5) the fallback waits for full write-out instead of guessing (`snapshotcache.go:pollWritebackFiles`).
- Identity checks (`os.SameFile` plus size) guard both drop and evict paths (`snapshotcache.go:DropFile`, `snapshotcache.go:EvictFiles`).

## Side Effects

- Unlinks staged files (`snapshotcache.go:DropFile`), syncs file ranges (`unix.SyncFileRange`), and issues `FADV_DONTNEED` evictions (`snapshotcache.go:EvictFiles`); sleeps between passes.

## Verification

- Run: `go test ./internal/snapshotcache/...`
