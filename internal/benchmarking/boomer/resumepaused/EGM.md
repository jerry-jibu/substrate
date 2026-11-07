# resumepaused

## Overview

Drives ate-api-server's cold resume path: every ResumeActor targets a suspended actor, so ateapi takes the actor lease, binds a worker, and waits on an atelet restore; each resume is followed by a suspend that makes the actor cold again. On the fake data plane it measures what a cold resume costs ateapi and Postgres, paced to `target_rps` cold resumes per second. Sends the actors themselves no requests.

## Interface

- `resumepaused.go:init` registers `userclass.Entry{Name: "resumepaused", UserClass: "ResumePausedUser", Config: knobsCodec, Init: initResumePaused}`.
- `resumepaused.go:knobs` - `TargetRPS` (`target_rps`), validated non-negative by `resumepaused.go:knobsCodec`.
- `resumepaused.go:runtime.iterate()` - the boomer task: first iteration creates and warms one actor per user (`rp-<uuid>` from the `benchmark-workloads/glutton` template), later ones run a paced cycle.
- `resumepaused.go:runtime.cycle(name)` - one paced `ResumeActorCold` then `SuspendActorAsync`.
- `resumepaused.go:runtime.shutdown(ctx)` - waits for in-flight cycles, then deletes every actor 16-way parallel with per-delete retries.
- `resumepaused.go:runtime.call(ctx, name, do)` - times one unary RPC, records client latency and (on success) server latency from the `ateinterceptors.ServerElapsedTrailer` trailer.

## Invariants

- Only the resume is paced (burst 1), so the target rate is cold resumes per second (`resumepaused.go:cycle`, `resumepaused.go:pace`).
- A `ResumeActor` whose response reports `resumed == false` fails the sample with `errAlreadyRunning`: it was not a cold resume (`resumepaused.go:resume`).
- A failed cycle recovers with an async suspend, so the actor is cold for the next one; a FailedPrecondition there counts as success since the actor is already cold (`resumepaused.go:recover`).
- Shutdown gives in-flight cycles half the remaining deadline, then cancels their context; `DeleteActor` removes an actor in any state so nothing is left behind (`resumepaused.go:shutdown`).
- Deletes retry Aborted and Unavailable up to 5 attempts (`resumepaused.go:delete`).
- Once shutdown begins, samples go to Prometheus only, because boomer may no longer drain its stats channel (`resumepaused.go:record`).
- Server-latency samples are recorded only for successful calls: a cut-off call brings back no trailer (`resumepaused.go:call`).

## Side Effects

- Calls ateapi RPCs: `CreateAtespace` (AlreadyExists is success), `CreateActor`, `ResumeActor`, `SuspendActor` (async), `DeleteActor`.
- Records locust stats rows (`ResumeActorWarmup`, `ResumeActorCold`, `SuspendActorAsync`, `SuspendActorAsyncWarmup`, `SuspendActorAsyncRecovery`, `CreateActor`, `DeleteActor`) and Prometheus histograms via `bmetrics`.
- Exits the process on a failed startup (atespace creation) (`resumepaused.go:initResumePaused`).

## Verification

- Run: `go test ./internal/benchmarking/boomer/resumepaused/...`
