# kubectl-ate/internal/printer

## Overview
Output renderers for `kubectl ate`. Every resource has a `Print<Resource>sTo(out, ..., format)` function emitting table, json, or yaml; json and yaml emit the bare API object.

## Interface
- `printer.go:PrintActorsTo` / `PrintActorTo`, `PrintWorkersTo` / `PrintWorkerTo`, `PrintActorTemplatesTo`, `PrintTagsTo`, `PrintAtespacesTo`, `PrintWorkerTopTo` - resource printers taking an `io.Writer` and format string.
- `printer.go:WorkerOccupancy` - renders a worker's actor count against its limit.
- `printer.go:TimeNow` - exported package variable for the clock, so tests can pin age rendering.
- `access_policy.go:PrintAccessPolicyTo` - one table row per binding; the caller passes the atespace (empty means global) because the policy carries no metadata.
- `egress_policy.go:PrintEgressPolicyTo` - one actor's egress policy; the caller passes the actor name.

## Invariants
- Table output sorts actors, workers, and atespaces deterministically (`printer.go:sortActors`, `sortWorkers`).
- Ages render kubectl-style ("5m", "3h", "2d") from creation timestamps via `printer.go:formatAge` and `TimeNow`.
- Nil capacity renders as "-" in worker tables (exercised by `printer_test.go:TestPrintWorkersTo_Table_Free`); the printer must not dereference a legacy CRD template ref.

## Side Effects
- None beyond writing to the provided `io.Writer`.

## Verification
- Run: `go test ./cmd/kubectl-ate/internal/printer/...`
