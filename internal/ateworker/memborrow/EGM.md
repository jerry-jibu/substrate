# memborrow

## Overview

Lends snapshot writes the worker pod's unbooked free memory for their page cache, and holds back new actor placements until that memory is written back. Built on `internal/ateworker/cgroup.PodMemory` and a caller-supplied `Unbooked` capacity function.

## Interface

- `memborrow.go:Ledger` - the borrow ledger. Fields `Unbooked func(podLimit int64) int64` and `PodMemory` (test override for `cgroup.PodMemory`).
- `memborrow.go:Ledger.Take(want) int64` - borrows up to `want`; the caller passes what it got to `Return`.
- `memborrow.go:Ledger.TakeAll(want) bool` - borrows all of `want` or nothing.
- `memborrow.go:Ledger.Return(n)` - gives back borrowed bytes.
- `memborrow.go:Ledger.Wait(ctx) error` - blocks a newly hosted actor until borrowed bytes fit within unbooked capacity.

## Invariants

- A nil `Ledger` lends nothing and waits for nothing (`memborrow.go:Take`, `memborrow.go:Wait`).
- A borrow is capped at the smaller of unbooked capacity minus current borrow, and pod limit minus unreclaimable memory minus a 32MiB margin (`memborrow.go:take`, `memborrow.go:margin`).
- Without a pod memory limit (`memory.max` = max), `Take` returns all of `want` (`memborrow.go:take`).
- `TakeAll` holds none of the memory when it cannot serve all of it (`memborrow.go:TakeAll`).

## Side Effects

- Reads worker pod cgroup memory via `cgroup.PodMemory()` (`memborrow.go:podMemory`).
- Logs "Waiting for snapshot writeback before booting" with borrowed bytes (`memborrow.go:Wait`).

## Verification

- Run: `go test ./internal/ateworker/memborrow/...`
