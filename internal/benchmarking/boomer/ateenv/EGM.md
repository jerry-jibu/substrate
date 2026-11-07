# ateenv

## Overview

Boomer-Go user class for the ate-env benchmark workload (`AteEnvUser`, locust stub `ate_env.py`). Each virtual user creates `ActorsPerUser` actors from the `benchmark-workloads/ate-env` template and cycles them through resume, an in-sandbox `echo hello world` executed over the ateenv gRPC API through the atenet router, a live window, and hibernate (async suspend, sync suspend, or none per `lifecycle_mode`).

## Interface

- `lifecycle.go:init` registers `userclass.Entry{Name: "ate_env", UserClass: "AteEnvUser", Config: ateEnvCodec, Init: initAteEnv}`.
- `lifecycle.go:ateEnvKnobs` - `min_live_time`, `max_live_time`, `max_pings_per_wake` plus embedded `dynconfig.WaitTime` and `dynconfig.Lifecycle`, validated by `lifecycle.go:ateEnvCodec`.
- `lifecycle.go:taskRuntime.iterate()` - the boomer task fn: one resume/exec/hibernate cycle per call; `lifecycle.go:taskRuntime.shutdown(ctx)` suspends and deletes every actor and closes router connections.
- `lifecycle.go:ateEnvActor` - per-actor state and lifecycle calls: `create`, `resume` (first resume recorded as `ResumeActorFirstResume`), `pause`/`suspend`, `delete`, `execHelloWorld`, `hibernate`.
- `lifecycle.go:routerGRPCTarget(rawURL)` - converts an HTTPS router URL to a `dns:///` gRPC target (https only).
- `lifecycle.go:ateEnvUser` - one VU holding a round-robin of actors; `replaceActor` swaps a broken one.

## Invariants

- `execHelloWorld` requires a non-empty process ID, an exit event with state `PROCESS_STATE_EXITED` and code 0, and stdout exactly `"hello world\n"` (`lifecycle.go:execHelloWorld`).
- Exec timeouts are 30s; Canceled, ResourceExhausted, InvalidArgument, PermissionDenied, Unauthenticated, and Unimplemented exec failures do not count toward replacement (`lifecycle.go:noteExecFailure`, `lifecycle.go:execTimeout`).
- Three consecutive lifecycle failures replace the actor (`lifecycle.go:maxConsecutiveFailures`, `lifecycle.go:countFailure`); a crashed actor stops receiving requests until replaced (`lifecycle.go:resume`).
- A failed hibernate leaves `hibernatePending`; the next iteration finishes the re-entrant suspend before resuming (`lifecycle.go:iterate`).
- A shutdown stops new iterations first and waits for in-flight ones, capped at half the remaining deadline so cleanup still runs (`lifecycle.go:shutdown`).
- A partial actor-creation failure keeps the actors created so far; zero actors fails the start (`lifecycle.go:startUser`).

## Side Effects

- Calls ateapi control RPCs: `CreateAtespace` (AlreadyExists is success), `CreateActor`, `ResumeActor`, `SuspendActor` (sync or async per `lifecycle_mode`), `DeleteActor` (`lifecycle.go`).
- Dials the atenet router over TLS and calls `ateenvv1alpha.ProcessService.StartProcess` and `StreamProcessOutput` with the `atenet.TargetActorHeader` metadata (`lifecycle.go:execHelloWorld`).
- Records locust stats rows (`ResumeActor`, `ResumeActorFirstResume`, `SuspendActor`, `SuspendActorAsync`, `DeleteActor`, `AteEnvExec`, `CrashCount`) via `bmetrics` (`lifecycle.go:tracedCall`).

## Verification

- Run: `go test ./internal/benchmarking/boomer/ateenv/...`
