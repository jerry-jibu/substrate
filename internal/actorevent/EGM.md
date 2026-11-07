# actorevent

## Overview
Defines the actor event vocabulary (`StateChanged`, `Crashed`, `UsageSampled`) and emits each event twice: as an OTLP log record under a fixed instrumentation scope, and as a stdout slog record. Also provides an async slog handler so a slow stdout cannot block the caller.

## Interface
- `actorevent.go:ScopeName` - instrumentation scope constant for every actor event.
- `actorevent.go:Event` - event descriptor: name, stdout `Level()`, promised `Keys`, and `Conditional` keys.
- `actorevent.go:BuildRecord` - converts a stdout record into its OTLP `log.Record` form.
- `actorevent.go:Emitter`, `actorevent.go:NewEmitter`, `actorevent.go:NewEmitterTo` - logger-bound emitter; `LogAt` writes at a caller-supplied time, `Log` at `time.Now()`.
- `actorevent.go:Log`, `actorevent.go:LogAt` - package-level emission through the process-wide provider.
- `actorevent.go:SetConsole` - toggles writing the stdout record beside the OTLP event.
- `async.go:AsyncHandler`, `async.go:NewAsyncHandler` - `slog.Handler` that writes from its own goroutine; `Close` drains and stops the writer.

## Invariants
- `Conditional` keys on an `Event` are present exactly when the registry's condition holds; absence means not measured, zero means measured as zero (`actorevent.go:Event`).
- `Emitter.emit` does nothing, cheaply, until a logs provider is installed; with none installed the event goes to stdout only (`actorevent.go:emit`).
- `BuildRecord` sets no trace context; the SDK lifts it from `ctx` onto the record (`actorevent.go:BuildRecord`).
- `events` is the whole vocabulary; an event left out is never checked against `docs/metrics/registry/events.yaml` (`actorevent.go:events`).
- `AsyncHandler.Handle` drops and counts a record when the queue is full, and the writer reports the drop count through the same handler once it catches up (`async.go:AsyncHandler`).

## Side Effects
- Emits OTLP log records through the configured `log.LoggerProvider` (`actorevent.go:Emitter.emit`).
- Writes stdout records through `slog.Default()` or the handler given to `NewEmitterTo` (`actorevent.go:LogAt`).
- `NewAsyncHandler` starts a writer goroutine; `Close` stops it after draining (`async.go:NewAsyncHandler`, `async.go:Close`).
- `SetConsole` mutates a package-global flag, meant to be called once at startup (`actorevent.go:SetConsole`).

## Verification
- Run: `go test ./internal/actorevent/...`
