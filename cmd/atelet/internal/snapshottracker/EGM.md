# snapshottracker

## Overview

Tracks each actor's snapshots on an atelet node, uploads and detaches them in
the background, and reports what those change to the control plane. The
tracker's contract (serialization of lifecycle RPCs, workqueue scheduling,
and the RPC vs background-work matrices) is documented in full on
`tracker.go:Tracker`.

## Interface

- `tracker.go:New` - returns a `Tracker` over a `Store` and clock; options `WithBootCheck`, `WithUploadDelay`, `WithDetachDelay`, `WithUploadTimeout`, `WithUploadEstThroughput`, `WithUploadWorkers`, `WithDetachWorkers`, `WithInstruments`, `WithReporter`, `WithReportWorkers`, `WithReportGiveUpAfter`.
- `tracker.go:Store` - seam for moving snapshots to object storage (`UploadLocalSnapshot`).
- `report.go:Reporter` - seam for reporting an actor's snapshot state (`ReportActorSnapshotState`).
- `actor.go:SnapshotRef` - one tracked snapshot reference (JSON-marshaled through `actor.go:(SnapshotRef).MarshalJSON`).
- `lifecycle.go:CommitLocal`, `CommitDurable`, `CommitPromoted`, `CommittedCheckpointState`, `LocalSnapshotsToKeep` - commit checkpoints and promoted snapshots, report committed state.
- `lifecycle.go:BeginStart`, `BeginCheckpoint`, `BeginPromote`, `BeginTerminate`, `Forget`, `NotStartedSince` - lifecycle gates that wait out or cancel in-flight background work per actor.
- `detach.go:(*Tracker).Detach` - foreground detach: makes LatestLocal durable, deletes the actor's local snapshots and files.
- `tracker.go:(*Tracker).State` - returns the actor's snapshot state.
- `background.go:(*Tracker).Start`, `background.go:(*Tracker).Wait` - starts the worker pools; waits for them to drain.
- `localcheckpoints.go:PruneLocalCheckpoints` - deletes local snapshots except the named ones.
- `metrics.go:Instruments`, `metrics.go:NewInstruments` - OTel histograms/counters for upload and detach duration and lateness, overdue-actor tally, oldest undurable snapshot age (nil-safe).

## Invariants

- Callers serialize lifecycle RPCs (Run, Restore, Checkpoint, PromoteSnapshot, DetachActor, Terminate) per actor before calling Tracker; at most one runs for an actor at a time (`tracker.go:Tracker` doc).
- BeginCheckpoint and BeginPromote reject a stale generation before Checkpoint or PromoteSnapshot does any work; commits never replace a snapshot with an older generation (`tracker.go:Tracker` doc).
- Uploads and detaches drain two separate delaying workqueues with separate worker pools (defaults 2 each), so slow uploads cannot hold up a due detach, nor detaches an upload (`tracker.go:Tracker` doc, `background.go:DefaultUploadWorkers`).
- An actor has at most one background operation in progress (`actorOp`); a worker that finds one leaves the actor to that operation's end (`tracker.go:Tracker` doc).
- A worker uploads the LatestLocal of when it runs, so snapshots superseded while waiting are never uploaded; an upload that has begun goes on across newer checkpoints and restarts until it succeeds or is superseded durably (`tracker.go:Tracker` doc).
- A Run or Restore unschedules the actor's detach as it begins and saves that before the workload can start, so atelet never detaches an actor whose start has begun (`tracker.go:Tracker` doc, `lifecycle.go:BeginStart`).
- Per-actor state persists to `ateletpath.ActorSnapshotStateFile` (snapshot-state.json) via `atomicfile.Write`, best-effort: on failure the entry stays in memory and a restart loses what changed (`state.go:writeStateLocked`).
- Reports retry with per-actor and queue-wide backoff until acknowledged or refused, canceled, or `DefaultReportGiveUpAfter` (10 minutes) passes; a detach's report being acknowledged (or refused NotFound) removes the actor's snapshot state file and directory, on give-up the state file stays so Detach keeps returning LatestDurable (`report.go`, `tracker.go:Tracker` doc).
- Saved reports recovered at startup are spread over `reportStartupJitter` (30s) so a fleet-wide restart does not hit the control plane at once (`report.go:reportStartupJitter`).
- A failure holds an actor's background work back `snapshotRetryDelay` (1 minute), doubling per consecutive failure up to `maxSnapshotRetryDelay` (10 minutes); each failure also lengthens the next upload attempt's timeout, doubling to at most 8x (`background.go`, `tracker.go:WithUploadTimeout`).
- With `WithBootCheck`, local snapshots written before the last node boot are deleted (their files may be torn) (`tracker.go:WithBootCheck`).
- Defaults: upload delay 5 minutes, detach delay 30 minutes, upload timeout 2 minutes over `DefaultUploadEstThroughput` (100 MiB/s) (`background.go`).

## Side Effects

- Reads and writes `snapshot-state.json` per actor and deletes local snapshot directories and actor files on detach (`state.go`, `detach.go`, `localcheckpoints.go:PruneLocalCheckpoints`).
- Uploads snapshots to object storage through `Store.UploadLocalSnapshot`; reports state to the control plane through `Reporter.ReportActorSnapshotState`.
- Runs background goroutines (upload, detach, and report worker pools) until the tracker's context ends; emits OTel metrics and structured logs.

## Verification

- Run: `go test ./cmd/atelet/internal/snapshottracker/...`
