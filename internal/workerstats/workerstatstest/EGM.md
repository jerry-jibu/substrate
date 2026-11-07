# workerstatstest

## Overview

Records the usage records a worker writes, for tests. `Recorder` is a
`slog.Handler` that captures the kind and source attributes of each
`ate.actor.usage_sampled` record.

## Interface

- `NewEmitter() (*workerstats.UsageEmitter, *Recorder)` (`workerstatstest.go:NewEmitter`): an emitter that writes every record to a new Recorder.
- `Recorder` (`workerstatstest.go:Recorder`): implements `slog.Handler` (`workerstatstest.go:Handle`).
- `(*Recorder).Kinds() []string` (`workerstatstest.go:Kinds`): the `ate.stats.kind` of each record, in order.
- `(*Recorder).Sources() []string` (`workerstatstest.go:Sources`): the `ate.stats.source` of each record, in order.

## Invariants

- Records are kept in the order they were handled (`workerstatstest.go:Kinds` doc).
- Safe for concurrent use; guarded by a mutex (`workerstatstest.go:Recorder`).
- Only the `ate.stats.kind` and `ate.stats.source` attributes are captured (`workerstatstest.go:Handle`).

## Side Effects

- None. In-memory recording only.

## Verification

- Run: `go test ./internal/workerstats/workerstatstest/...`
