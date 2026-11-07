# workerstats

## Overview

Usage telemetry both worker runtimes share: the attribution a worker retains
for each actor, the per-activation state behind every sample, the sampler, and
the `ate.actor.usage_sampled` records. The per-runtime measurement reads live
with their runtimes (`actorattribution.go` package comment).

## Interface

- `Pool` (`usage.go:Pool`): the WorkerPool identity stamped on records.
- `UsageEmitter` / `NewUsageEmitter(lp *sdklog.LoggerProvider, stdout slog.Handler, pool Pool) *UsageEmitter` (`usage.go:NewUsageEmitter`): writes `ate.actor.usage_sampled` records over OTLP and/or stdout.
- `(*UsageEmitter).Emit(ctx, kind string, s *ateworkerpb.WorkloadStatsSample)` (`usage.go:Emit`): writes one record dated when the sample was observed.
- `UsageAttrs(pool, kind string, s) []slog.Attr` (`usage.go:UsageAttrs`): the record's attribute set.
- `StatsSourceLabel(s ateworkerpb.StatsSource) string` (`usage.go:StatsSourceLabel`): maps the wire enum to `ate.stats.source` values.
- `NewStdoutHandler(w io.Writer) *actorevent.AsyncHandler` (`usage.go:NewStdoutHandler`): stdout JSON form of the records, off the caller's goroutine.
- `StartSampler(ctx, interval, sweep func(context.Context)) (stop func())` (`usage.go:StartSampler`): periodic sampler on its own goroutine.
- `ActorAttributionFromRequest(req) resources.ActorAttribution` (`actorattribution.go:ActorAttributionFromRequest`): extracts actor attribution from Run/Restore/Checkpoint/Terminate workload requests (`actorattribution.go:attributionSource` assertions).
- `Activation` / `NewActivation(now time.Time, resumesCounters bool) *Activation` (`activation.go:NewActivation`): per-activation usage state, safe for concurrent use.
- `(*Activation)` methods: `Epoch`, `WithEpoch`, `Measure(ctx, read)` (`activation.go:Measure`), `Store`, `Initial(s, write)`, `Sampling()`, `Periodic(s, write)`, `Final(write)`, `Latest()` (`activation.go:Latest`).
- Subdirectory:
  - [workerstatstest](./workerstatstest/EGM.md) - Recorder capturing the kind and source of each usage record for tests.

## Invariants

- The accumulated CPU total never decreases: each counter's increase accumulates, a missing counter keeps its last value, a decreased counter (restart) counts its new value in full, and an unseen counter counts in full only when the activation's counters start at zero (`activation.go:fold`).
- Counter addition saturates at `math.MaxUint64` so a guest-reported counter cannot wrap the total (`activation.go:addSat`).
- Periodic records are written only between the initial reading and the final record, so the initial record comes first and the final one last (`activation.go:Sampling`, `activation.go:Periodic`).
- `Initial` and `Final` take effect once; later calls are no-ops (`activation.go:Initial`, `activation.go:Final`).
- The `write` callbacks run under the activation's lock and must not block or call into the activation (`activation.go:Initial` doc).
- Readings are serialized by a one-slot channel lock; a caller waiting for an in-flight reading gives up with its context (`activation.go:Measure`).
- Callers must not modify the sample returned by `Latest` (`activation.go:Latest` doc).
- A nil `*UsageEmitter` writes nothing (`usage.go:UsageEmitter` doc).
- Measurements are absent while the sample has no source, and the peak is absent when the source reports none (`usage.go:UsageAttrs`).
- The stdout handler logs at a fixed Info level so `--log-level` cannot silence it, with a 4096-record queue (`usage.go:NewStdoutHandler`, `usage.go:stdoutQueue`).
- A panicking sweep is recovered and logged, so a sampler bug cannot take the worker and its actors down (`usage.go:sweepOnce`).

## Side Effects

- Emits OTLP log records and/or stdout JSON lines for each sample (`usage.go:NewUsageEmitter`, `usage.go:Emit`).
- Runs a ticker-driven sampler goroutine; the returned stop function cancels it and waits for the in-flight sweep (`usage.go:StartSampler`).
- Accumulates CPU totals and the latest samples in memory per activation (`activation.go:Activation`).

## Verification

- Run: `go test ./internal/workerstats/...`
