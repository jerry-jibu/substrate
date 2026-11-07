# fspersist

## Overview

A demo actor workload proving the filesystem persists across
checkpoint/restore. On each request it prepends a line recording the current
pod IP and a running count to a history file on the actor's root filesystem
(capped at 20 lines) and returns the file. The count is derived from the
persisted file, not memory, so it accumulates across suspend/resume and the
recorded pod IPs reveal each move to a new worker (`fspersist.go:15`-`:27`).

## Interface

- `main` (`fspersist.go:54`): HTTP server on port 80; `/` returns the pod IP
  plus the persisted history (`fspersist.go:61`-`:77`).
- `appendHistory` (`fspersist.go:90`): reads, prepends to, and rewrites
  `/pod-history.log` (`historyPath`, `fspersist.go:46`), capped at
  `maxLines` 20 (`fspersist.go:49`).
- `nextCount` (`fspersist.go:124`) derives the count from the newest history
  line; `getCurrentIP` (`fspersist.go:139`) returns the first non-loopback
  IPv4 address.

## Invariants

- The count is read back from the persisted file rather than memory, so it
  survives checkpoint/restore (`fspersist.go:86`-`:89`, `:121`-`:123`).
- History writes serialize through `fileMu` (`fspersist.go:52`, `:91`).
- The history file keeps at most 20 lines, newest first (`fspersist.go:107`-`:110`).

## Side Effects

- Reads and rewrites `/pod-history.log` on the actor's root filesystem on
  every request (`fspersist.go:97`, `:113`).

## Verification

- Run: `go test ./demos/...`
