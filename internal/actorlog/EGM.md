# actorlog

## Overview
Structured logging for actor sandbox output and lifecycle events. Wraps actor container log lines into a unified JSON format stamped with substrate-owned identity labels, and emits synthetic lifecycle records that join the driving RPC's trace.

## Interface
- `logger.go:ActorLogger`, `logger.go:NewActorLogger` - wraps a destination `io.Writer`; `isOnGCE` selects the label-group spelling Cloud Logging promotes.
- `logger.go:WrapContainerLogs` - reads lines from a reader, parses plain or JSON input, and logs each in the unified format tagged with `containerName`.
- `logger.go:StartJSONLogPipe` - returns an `io.WriteCloser` whose writes are logged through the logger.
- `logger.go:EmitLifecycleLog` - logs a synthetic actor lifecycle event, joining the RPC trace on `ctx`.
- `logger.go:SyncedWriter`, `logger.go:NewSyncedWriter` - `io.Writer` that synchronizes writes across goroutines.

## Invariants
- Plain-text actor lines log with no level; levels are never guessed (`logger.go:WrapContainerLogs`).
- Label identity comes from `internal/ateattr` spellings (`ateattr.ActorLogLabels`); the actor cannot forge substrate's namespace or a foreign `logging.googleapis.com/labels` group that would outrank substrate attribution (`logger.go:ActorLogger`).

## Side Effects
- Writes JSON log records to the destination writer (`logger.go:NewActorLogger`).
- `WrapContainerLogs` reads the container's output stream until EOF and mutates nothing the caller keeps.

## Verification
- Run: `go test ./internal/actorlog/...`
