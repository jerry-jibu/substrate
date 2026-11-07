# report

Holds what a benchmark run measured and prints it. Pure data types plus the
table renderer and the JSON writer; no measurement logic.

## Interface

- `report.go:Percentiles` - latency summary in milliseconds: p50, p90, p99, max, mean.
- `report.go:Load` - what the generator saw during one rung: `Offered`, `MeasuredSeconds`, `Scheduled`, `Skipped`, `Completed`, `Errors`, `Achieved`, `Latency`, `SendLag`, `Bytes`, `MBytesPerSec`, `ErrorSamples`.
- `report.go:Resources` - one process's cost over the measured window: CPU seconds and cores, `MicrosPerUnit`, GOMAXPROCS, RSS and heap end and max, goroutine peak, live-after-GC figures, and cgroup v2 fields when in a container.
- `report.go:Rung` - one offered rate: `Load`, `Tunnel` (atunnel, process B), `Generator` and `Sink` (processes A and C), `Verdict`, `Reason`, `Profiles`.
- `report.go:Report` - one scenario across its rungs, with `SinkCounters` for checking against the generator.
- `report.go` verdict constants - `VerdictOK`, `VerdictRigLimited`, `VerdictErrors`, `VerdictSaturated`, `VerdictDown`, `VerdictShortfall`.
- `report.go:Report.String` - renders the report as a text table, adding cgroup columns when any rung has them and a throughput or per-connection line where it applies.
- `report.go:Write` - saves reports to a path as indented JSON.

## Invariants

- Verdict strings are fixed constants; the ladder in `internal/load` sets them and `String` prints them verbatim (`report.go` verdict constants).
- `Write` marshals with `json.MarshalIndent` and writes mode 0644 (`report.go:Write`).

## Side Effects

- `Write` creates or truncates the output file (`report.go:Write`).

## Verification

- Run: `go test ./benchmarking/atunnel/internal/report/...` from the repo root.
