# sweperf

## Overview

Boomer-Go implementation of the SweperfUser locust test. Each user creates an actor from a SWE-bench workload template (default `benchmark-workloads/sweperf-astropy-7336`) and drives it through a trajectory of steps partitioned into cycles: resume the actor, POST the cycle's step range to the in-sandbox `/execute` endpoint through the atenet router, poll `/status?job_id=` until the job completes, then suspend. When the trajectory is done the actor is deleted and a fresh one runs it again.

## Interface

- `sweperf.go:init` registers `userclass.Entry{Name: "sweperf", UserClass: "SweperfUser", Config: knobsCodec, Init: initSweperf}`.
- `sweperf.go:knobs` - `sweperf_template`, `sweperf_total_steps`, `sweperf_num_cycles`, `sweperf_poll_interval_ms` plus embedded `dynconfig.WaitTime`/`Lifecycle`; validated by `sweperf.go:knobsCodec`.
- `sweperf.go:generateDynamicChunks(totalSteps, numCycles) []chunk` - contiguous cycle partition, lengths differing by at most one.
- `sweperf.go:sweperfRuntime.iterate()` / `sweperf.go:sweperfRuntime.shutdown(ctx)` - boomer task and teardown hooks.
- `sweperf.go:sweperfUser.step(ctx)` - one resume/execute/suspend cycle; the final cycle skips the suspend.
- `sweperf.go:sweperfUser.execute(ctx, cycleNum, startIdx, endIdx)` - POSTs `/execute`, handles sync (`exit_code`) and async (`job_id`) replies; `sweperf.go:sweperfUser.pollLiveness(ctx)` and `sweperf.go:sweperfUser.pollJobCompletion(ctx, jobID, cycleNum)` poll `/status`.
- `sweperf.go:sweperfUser.tracedCall`/`tracedCallRecord` - RPC wrapper preferring server trailer latency, publishing client RTT as `<name>_rtt` rows.
- Derived rows under method `actor`: `ResumeToFirstExec`, `CycleCEL`, `TaskCEL`, `TaskWallClock` (`sweperf.go` metric constants), plus `Workload_Cycle_<n>` HTTP rows.

## Invariants

- `lifecycle_mode "none"` and `resume_mode "implicit"` are refused: every sweperf cycle ends in a suspend and resumes with ResumeActor (`sweperf.go:knobsCodec.Validate`).
- An unset knob falls back to the built-in defaults: template `sweperf-astropy-7336`, 21 steps, 4 cycles, 100ms poll interval (`sweperf.go:resolveConfig`, `sweperf.go:pollInterval`).
- `CreateActor` runs detached from the run context: canceled mid-call, the actor might exist without the session knowing to delete it (`sweperf.go:startUser`, `sweperf.go:teardownCtx`).
- Actor deletion happens exactly once: `cleanedUp.CompareAndSwap` between the owning goroutine and shutdown (`sweperf.go:retireUser`, `sweperf.go:sweperfUser.teardown`); no suspend is sent first because DeleteActor removes any state.
- A no-op resume (actor left running by pollLiveness) records no ResumeActor success sample and no ResumeToFirstExec near-zero sample (`sweperf.go:sweperfUser.step`, `sweperf.go:sweperfUser.resume`).
- A cycle cut short by shutdown records nothing (`canceled(ctx)` checks in `sweperf.go:sweperfUser.step`, `sweperf.go:httpJSONCall`).
- Job polling fails on non-zero exit code, FAILED status, or a 2-minute timeout; only COMPLETED with exit 0 succeeds (`sweperf.go:pollJobCompletion`).
- Shutdown waits at most 10s for in-flight iterations, then sweeps actors 64 at a time; a mid-run decrease in user count leaks actors until shutdown because boomer has no per-VU stop hook (`sweperf.go:shutdown`).

## Side Effects

- Calls ateapi RPCs: `CreateAtespace` (AlreadyExists is success), `CreateActor`, `ResumeActor`, `SuspendActor` (async by default, sync in sync mode), `DeleteActor`.
- POSTs `/execute` and GETs `/status` on the actor's in-sandbox server through the atenet router with the `atenet.TargetActorHeader` header (`sweperf.go:httpJSONCall`, `sweperf.go:pollLiveness`).
- Records locust stats rows and a `sweperf task record` structured log line per finished trajectory (`sweperf.go:logTaskRecord`).
- FailedPrecondition from CreateActor clears `atespaceReady` so the next start recreates the atespace (`sweperf.go:startUser`).

## Verification

- Run: `go test ./internal/benchmarking/boomer/sweperf/...`
