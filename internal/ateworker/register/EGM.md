# register

## Overview

Registers a worker process with the control plane through the node-local atelet: acquires an epoch, registers once with that epoch, its pod IPs and its compute capacity, then keeps its sandbox runtimes reported whenever atelet rewrites the runtimes file. Also provides file-watching and crash-report helpers shared by worker binaries.

## Interface

- `register.go:Epoch` - the registered epoch; `Load`, `Store`, `Registered(ctx)` (readiness check), `Require()` (Unavailable before registration).
- `register.go:FromFiles(actors int, memoryReserved int64) (*ateletpb.WorkerResources, error)` - combines downward API compute limits with the worker's actor limit.
- `register.go:ParsePodIPs(value) []string` - splits the downward API `status.podIPs` value.
- `register.go:ReportConfig` - socket path, credential/trust bundle paths, `AteletSPIFFEID`, `Actors`, `PodIPs`, `MemoryReservedBytes`, `ActorOverhead`, `SandboxClass`, `RuntimesFile`.
- `register.go:Report(ctx, cfg, epoch) error` - registers and reports runtimes until ctx ends.
- `register.go:supportClient` - the slice of atelet's `WorkerSupport` used: `AcquireWorkerEpoch`, `RegisterWorker`, `SetWorkerSandboxRuntimes`.
- `watch.go:WatchFilePath(ctx, path) <-chan struct{}` - signals when the file is created, written, or replaced.
- `crash.go:NewCrashReporter(cfg ReportConfig)` / `crash.go:CrashReporter.ReportCrashed(ctx, actor resources.ActorAttribution, message string) error`.

## Invariants

- A worker takes no workloads until registered: `Epoch.Registered` fails and `Epoch.Require` returns Unavailable while the epoch is 0 (`register.go:Epoch`).
- `Report` requires `SandboxClass`, `RuntimesFile`, and at least one pod IP; configuration errors return early because no retry would fix them (`register.go:Report`).
- An unreadable or unparseable capacity limit is reported as zero, which the control plane reads as none (`register.go:readLimit`).
- A pod memory limit no greater than `memoryReserved` is a startup error (`register.go:fromDir`).
- A registration that timed out is resent with the same epoch, which the control plane accepts again without a write; a `FailedPrecondition` refusal of the epoch ends the process's serving (`register.go:registerOnce`).
- The reconciler reports runtimes only when they differ from the last accepted request (`proto.Equal` in `register.go:reconciler.sync`), and change signals are ignored while backing off so rapid writes cannot bypass backoff (`register.go:reconciler.run`).
- Crash messages are truncated to 4096 bytes on a rune boundary (`crash.go:truncateMessage`); a crash-report refusal (NotFound, FailedPrecondition) is an answer and is logged, not returned (`crash.go:reportCrashedWithRetry`).
- `WatchFilePath` watches the parent directory so atomic renames are caught, and falls back to 5s polling when fsnotify is unavailable (`watch.go:WatchFilePath`).

## Side Effects

- Dials the node-local atelet over mTLS (via `ateletdial.Dial`) with a 10s per-call timeout (`register.go:dialingClient`).
- Calls atelet RPCs: `AcquireWorkerEpoch`, `RegisterWorker`, `SetWorkerSandboxRuntimes`, `ReportActorCrashed`.
- Reads downward-API limit files under `workercapacity.MountPath` and the sandbox runtimes file (`register.go:FromFiles`, `register.go:reconciler`).
- Starts fsnotify watchers or polling tickers in goroutines (`watch.go:WatchFilePath`).
- Emits structured logs for registration, runtime reports, retries, and warnings.

## Verification

- Run: `go test ./internal/ateworker/register/...`
