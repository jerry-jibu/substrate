# glutton

## Overview

The benchmarking workload served by `cmd/benchmarking/glutton`: an in-sandbox API for consuming RAM, disk, file descriptors, and CPU, plus peer gossip, over gRPC or protobuf-over-HTTP (`internal/proto/glutton`). The routes and service are shared between wire modes; the fake stand-in lives in `./fake`.

## Interface

- `glutton.go:Name` ("glutton"), wire modes `glutton.go:ModeGRPC`/`ModeHTTP`, and route constants `ReadyzRoute`, `PingRoute`, `WriteDiskRoute`, `ReadDiskRoute`, `WriteRAMRoute`, `ReadRAMRoute`, `BurnCPURoute`, `IngestRoute`, `UseCPURoute`, `RunScriptRoute`.
- `server.go:Handler(mode, svc) (http.Handler, error)` - gRPC (with reflection and the readyz probe on one listener) or the otelhttp-wrapped HTTP route table.
- `server.go:NewServer(handler) *http.Server` - enables unencrypted HTTP/2 next to HTTP/1.1.
- `service.go:New(dir, opts...) (*Service, error)` with options `service.go:WithSyncWrites()` and `service.go:WithWritableReadyz()`; `service.go:Service.Close()`.
- `service.go` RPCs (implement `gluttonpb.GluttonServer`): `WriteRAM`, `ReadRAM`, `WriteDisk`, `ReadDisk`, `OpenFD`, `Ping`, `BurnCPU`, `Ingest`; `gossip.go:Service.Gossip`; `cpu_load.go:Service.UseCPU`; `script.go:Service.RunScript`.
- `sizes.go:parseBytes(s)` - plain bytes or Ki/Mi/Gi and K/M/G suffixes.
- Subdirectory: [fake](./fake/EGM.md) - httptest stand-in for a glutton actor.

## Invariants

- Disk keys must match `^[a-zA-Z0-9_-]+$` so nothing escapes the data dir or names a hidden file (`service.go:diskKeyRE`).
- `/readyz` is served in both modes because the worker blocks RestoreWorkload on it; with `WithWritableReadyz` it fails 503 until the data dir accepts the `.readyz_canary` write (`server.go:readyzMux`, `glutton.go` route doc).
- RAM rotate (`WRITE_MODE_OVERWRITE_ROTATE`) needs an existing array and advances a per-key cursor that wraps, so repeated rotates walk the whole array; the cursor rides along in snapshots (`service.go:rotateRAM`).
- ReadRAM walks one byte per 4KiB page, forcing every page resident before it returns (`service.go:ReadRAM`, `service.go:pageSize`).
- WriteDisk streams 1MiB chunks so a multi-gigabyte write never materializes in RAM; OVERWRITE keeps bytes beyond the new size and folds them into the digest (`service.go:streamRandomBytes`, `service.go:WriteDisk`).
- BurnCPU caps parallelism at NumCPU*4 and runs unsynchronized with the service mutex so burning never blocks other RPCs (`service.go:BurnCPU`).
- UseCPU clamps `num_cores` to GOMAXPROCS by default and each worker measures CPU time via `CLOCK_THREAD_CPUTIME_ID`, so throttling cannot fool the duty cycle (`cpu_load.go:cpuLoad.Set`, `cpu_load.go:cpuLoadWorker`).
- RunScript validates the tree shape before running anything; a loop budget is checked between passes only, and a failing operation request cancels its siblings and keeps its status code (`script.go:validateBlock`, `script.go:runBlock`, `script.go:runOperation`, `script.go:prefixError`).
- The fake in `./fake` aliases these route constants so the stand-in and the real mux cannot drift apart (`glutton.go` route doc).

## Side Effects

- Writes files under the data dir (WriteDisk, Ingest), optionally fsynced; opens/closes `/dev/null` FDs (OpenFD) (`service.go`).
- Holds RAM arrays and cursors in process memory; starts and stops CPU-load goroutine pools pinned to OS threads (`service.go`, `cpu_load.go`).
- Gossip dials peers over insecure gRPC and pings them on a ticker until cancelled; peers are diffed per request (`gossip.go:Gossip`, `gossip.go:runGossip`).
- Registers otel instruments: `glutton.ram.write.bytes`, `glutton.ram.read.bytes`, `glutton.disk.write.bytes`, `glutton.disk.read.bytes`, `glutton.ping.requests`, `glutton.gossip.requests.sent`, `glutton.gossip.latency`, plus gauges `glutton.fds.open`, `glutton.gossip.delay`, `glutton.cpu.workers` (`metrics.go:initMetrics`).
- HTTP mode sets the `x-server-elapsed-us` header on every reply, mirroring the control-plane gRPC trailer (`server.go:protoRoute`).

## Verification

- Run: `go test ./internal/benchmarking/glutton/...`
