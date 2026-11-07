# phaselog

## Overview

Renders the per-actor timing-breakdown log record every worker binary emits for each checkpoint and restore. Each worker times its own phases; this package holds the shared record shape so all workers' records carry the same identity, fidelity label, and key layout, and atelet's record joins to any of them.

## Interface

- `phaselog.go:Phase` - one timed step: `Name string`, `D time.Duration`.
- `phaselog.go:SnapshotPhaseAttrs(a resources.ActorAttribution, fidelity ateworkerpb.SnapshotFidelity, durationKey string, err error, phases []Phase) []slog.Attr` - builds the joinable record.
- `phaselog.go:LogSnapshotPhases(ctx, msg, a, fidelity, durationKey, err, phases)` - emits the record at info level.
- `phaselog.go:FidelityLogValue(fidelity ateworkerpb.SnapshotFidelity) string` - maps the worker wire enum onto shared fidelity label values.
- Constants `phaselog.go:RestoreDurationKey` (`ateworker.actor.restore.duration`) and `phaselog.go:CheckpointDurationKey` (`ateworker.actor.checkpoint.duration`).

## Invariants

- The duration keys are log record keys, not metric instruments; they extend atelet's `ate.actor.*.duration` histogram names (`phaselog.go:RestoreDurationKey` doc).
- An unrecognized fidelity maps to unknown rather than being stringified, so no wire value widens the value set readers key on (`phaselog.go:FidelityLogValue`).
- Zero-duration phases are skipped, not logged as instant (`phaselog.go:SnapshotPhaseAttrs`).
- Phase durations are logged as float seconds to match the atelet histograms' unit (`phaselog.go:SnapshotPhaseAttrs`).
- A failed operation still records its completed phases, marked with `error.type` (gRPC code); absence means success (`phaselog.go:SnapshotPhaseAttrs`).

## Side Effects

- Emits structured log records via `slog.LogAttrs` (`phaselog.go:LogSnapshotPhases`).

## Verification

- Run: `go test ./internal/ateworker/phaselog/...`
