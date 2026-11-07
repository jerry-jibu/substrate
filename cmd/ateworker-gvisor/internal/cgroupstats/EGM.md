# cgroupstats

## Overview
Reads resource usage out of a cgroup v2 directory ("Package cgroupstats", `cgroupstats.go`). Used by the gVisor worker to report CPU and memory usage for hosted actors.

## Interface
- `cgroupstats.go:Read(dir string) (Sample, error)` parses the standard cgroup v2 files in `dir`.
- `cgroupstats.go:Sample` carries the parsed usage values.

## Invariants
- Missing or unreadable individual counters are reported as absent, not zero: `cgroupstats.go:readUint` and `readKeyedUint` return an ok flag the caller must honor.

## Side Effects
- Reads cgroup v2 pseudo-files from disk only.

## Verification
- Run: `go test ./cmd/ateworker-gvisor/internal/cgroupstats/...`
