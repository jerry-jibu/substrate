# filecachegc

## Overview

Runs the eviction loop of a node-local `filecache` store: a byte budget,
evicted least recently used first, down to a low-water fraction of the
budget, never taking an entry a consumer still links. One loop per cache,
one flag set per cache under the cache's flag prefix, one set of metrics for
all of them labeled by cache name.

## Interface

- `filecachegc.go:Start` - validates the cache's flags, registers its size metric, and runs the eviction loop in the background when budget and period are set.
- `filecachegc.go:Store` - the seam the loop needs from a `*filecache.Store` (`TotalBytes`, `EvictUnused`).
- `filecachegc.go:Flags`, `filecachegc.go:NewFlags` - registers `--<prefix>-gc-period`, `--<prefix>-max-bytes`, `--<prefix>-gc-dry-run` on a pflag set; `Flags.Validate` rejects unactionable values.
- `filecachegc.go:Instruments`, `filecachegc.go:NewInstruments` - OTel instruments `ate.filecache.size`, `ate.filecache.evictions`, `ate.filecache.gc.passes`; nil-safe.
- `filecachegc.go:target` (unexported) - computes bytes to free: nothing within budget, otherwise down to the `lowWater` (0.9) fraction.

## Invariants

- A pass only evicts entries nothing links, LRU first, down to `lowWater` of the budget so passes do not fire on every tick at the budget edge (`filecachegc.go:lowWater`, `filecachegc.go:target`).
- A zero `max-bytes` means no budget: the cache only grows, and `Start` warns once (`filecachegc.go:Start`).
- A pass fails toward retention: a sizing failure skips the pass and retries next tick (`filecachegc.go:runPass`).
- A panicking pass is recovered and skipped; this janitor must not take atelet down and strand every actor on the node (`filecachegc.go:runPass`).
- A dry run retires nothing and must not count as evictions (`filecachegc.go:runPass`).
- Persistent shortfalls warn `shortfallWarnLimit` (3) times then every `shortfallReminderEvery` (12) passes, never a line per tick (`filecachegc.go:runPass`).
- The first pass runs immediately at startup so a node booting over budget does not wait a full period; passes are strictly serialized (`filecachegc.go:run`).

## Side Effects

- Deletes cache entries through `Store.EvictUnused` on a background goroutine (`filecachegc.go:run`).
- Registers an OTel observable callback that walks the store's entries on every collection (`filecachegc.go:observe`).
- Emits metrics and structured logs per pass.

## Verification

- Run: `go test ./cmd/atelet/internal/filecachegc/...`
