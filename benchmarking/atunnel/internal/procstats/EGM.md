# procstats

Reads what a benchmark process costs: CPU time, resident memory, Go heap,
goroutines, and, inside a cgroup v2 container, the container's own CPU and
memory accounting. Each benchmark process serves its snapshot over HTTP so the
load generator can read it per rung.

## Interface

- `procstats.go:Snapshot` - one instant of a process's counters: `Time`, `StartedAt` (identifies the process; a change means it was replaced), `CPUSeconds`, `RSSBytes` with `RSSIsPeak`, `HeapInUseBytes`, `GoTotalBytes`, `Goroutines`, `GOMAXPROCS`, and optional `Cgroup`.
- `procstats.go:Cgroup` - cgroup v2 accounting: `CPUSeconds`, `MemoryCurrentBytes`, `MemoryPeakBytes`, and `CPULimitCores` from `cpu.max` (0 means unlimited).
- `procstats.go:Read` - takes a snapshot of this process.
- `procstats.go:Handler` - serves `Read` as JSON under `process`, merged with whatever `extra` returns; `?gc=1` collects garbage first so heap figures are live.
- `rusage_unix.go:cpuSeconds` / `rusage_unix.go:rss` - getrusage CPU time, and current RSS from `/proc/self/status` on Linux or peak RSS from getrusage elsewhere (Linux and Darwin builds).
- `rusage_other.go:cpuSeconds` / `rusage_other.go:rss` - zero stubs for other platforms.

## Invariants

- `StartedAt` is captured once at package init, so it changes only when the process is replaced (`procstats.go:startedAt`, `procstats.go:Read`).
- `RSSBytes` is current on Linux and the lifetime peak elsewhere, and `RSSIsPeak` says which (`rusage_unix.go:rss`).
- A process in a host's root cgroup reports `Cgroup` nil, because the hierarchy root has no `memory.current`; a process in its own cgroup reports that one, from `/proc/self/cgroup` (`procstats.go:readCgroup`, `procstats.go:ownCgroup`).

## Side Effects

- Reads `/proc/self/cgroup`, `/proc/self/status`, and the cgroup v2 files `cpu.stat`, `memory.current`, `memory.peak`, and `cpu.max` (`procstats.go:ownCgroup`, `rusage_unix.go:rss`, `procstats.go:readCgroup`).
- `Handler` with `?gc=1` runs `runtime.GC` in the serving process (`procstats.go:Handler`).

## Verification

- Run: `go test ./benchmarking/atunnel/internal/procstats/...` from the repo root.
