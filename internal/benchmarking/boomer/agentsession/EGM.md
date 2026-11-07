# agentsession

## Overview

Implements the AgentSessionUser locust test: each virtual user is one coding-agent session replaying a scripted YAML step sequence against a glutton actor. After every step the actor is suspended for the step's think gap; the next step's first request wakes it through the atenet router (request parking), so the benchmark measures user-visible wake latency. A turnover mode (`agentsession_awake > 0`) instead keeps a fixed number of sessions awake and swaps them at `target_rps`.

## Interface

- `agentsession.go:init` registers the user class via `userclass.Add` (`userclass.Entry{Name: "agentsession", UserClass: "AgentSessionUser", Config: knobsCodec, Init: initAgentSession}`).
- `agentsession.go:knobs` - the `--agentsession-*` runtime config slice (`agentsession.go:knobsCodec` validates it): `Script`, `ScriptFile`, `ThinkScale`, `SnapshotSteps`, `PhaseIn`, `ResumeRetryBudget`, `Awake`, `TargetRPS`, plus embedded `dynconfig.Lifecycle`.
- `agentsession.go:runtime.iterate()` - the boomer task fn, one scripted step per call; `agentsession.go:runtime.shutdown(ctx)` sweeps sessions.
- `agentsession.go:sessionUser` - one session: actor name, step index, wake/hibernate state; methods `runStep`, `wakeParked`, `hibernate`, `suspendAndDelete`, `wakePing`, `recordTimeToWake`.
- `scriptfile.go:Script` - name, `MinActorMemory`, steps; `scriptfile.go:Decode(data)` parses YAML with `KnownFields(true)`; `scriptfile.go:Encode(s)` renders YAML; `scriptfile.go:FormatSize(n)`.
- `script.go:Step` - `Name` (keys locust row `Step_<Name>`), `Agent`, `Think`, `Ops`; `script.go:op`/`opKind` - ingest, burn_cpu, write_disk, read_disk_digest, read_disk_data, fill_ram, churn_ram, walk_ram, ping, dwell (`scriptfile.go:opSpecs`).
- `validate.go:Validate(s)` and `validate.go:Budgets(steps) Budget` - script invariants and declared RAM/disk.
- `scripts.go:Names()`, `scripts.go:DefaultScript` ("coding-session"), `scripts.go:readScript` (built-in embed or worker file), `scripts.go:scriptFS`.
- `turnover.go:conductor` - coordinates wakes/parks (`tick`, `join`, `settle`, `claim`, `halt`).

## Invariants

- A script must declare `min_actor_memory` at least its declared RAM plus disk, unique non-empty steps with positive think times, and no op that consumes a RAM array or file before an earlier step created it (`validate.go:Validate`).
- Ops are single-key maps; unknown kinds, unknown fields, and arguments a kind does not take are errors; keys must match `^[a-zA-Z0-9_-]+$` (`scriptfile.go:Decode`, `scriptfile.go:decodeOp`, `scriptfile.go:keyRE`).
- The driver refuses to start against a template whose memory limit is below the script's `min_actor_memory`; a refusal is cached for 30s (`agentsession.go:checkTemplateMemory`).
- Capacity errors (ResourceExhausted, Unavailable, router 503/504/429) never count toward actor replacement, only toward wake retries within `ResumeRetryBudget` (`agentsession.go:noteFailure`, `agentsession.go:isCapacityError`, `agentsession.go:retryWhileFull`).
- Three consecutive `ReplaceIfPersistent` step failures mark the actor broken; it is then deleted and a fresh session starts (`agentsession.go:maxConsecutiveStepFailures`, `agentsession.go:iterate`).
- Cleanup of a session's actor happens exactly once: `LoadAndDelete` claims the cleanup on both the shutdown sweep and the racing start side (`agentsession.go:iterate`, `agentsession.go:shutdown`).
- A session captures the script it started on; a knob change mid-run applies to new sessions only (`agentsession.go:runtime.loadScript`).
- Suspend before delete, because a worker only frees an actor it can account for (`agentsession.go:suspendAndDelete`).
- Turnover fill waits for one minute of registration quiet before swapping (`turnover.go:registrationQuiet`); the conductor halts permanently on a boomer stop (`turnover.go:conductor.halt`).

## Side Effects

- Calls ateapi control RPCs: `CreateAtespace`, `CreateActor`, `ResumeActor`, `SuspendActor`, `DeleteActor`, `GetActorTemplate` (`agentsession.go`).
- POSTs protobuf glutton RPCs through the atenet router with the `atenet.TargetActorHeader` header (`agentsession.go:postProto`).
- Records locust stats rows (`Step_<name>`, `WakeFirstTouch`, `TimeToWake`, `CrashCount`, control RPC names) via `bmetrics` (`agentsession.go:runStep`, `agentsession.go:tracedCall`).
- Registers Prometheus metrics `agentsession_burn_cpu_iterations_per_second` (`agentsession.go:burnRate`) and `agentsession_turnover_sessions{state}` (`turnover.go:turnoverSessions`).
- Reads script YAML from the embedded FS or the worker's filesystem; starts a conductor ticker goroutine and shutdown sweep goroutines (64-way) (`agentsession.go:initAgentSession`, `agentsession.go:shutdown`).

## Verification

- Run: `go test ./internal/benchmarking/boomer/agentsession/...`
