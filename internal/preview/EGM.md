# preview

## Overview

Manages preview feature gates enabled via the `--preview` flag. Gates are
process-wide; a binary calls `Init` exactly once before anything calls
`IsEnabled` or `AsMap`. A gRPC request-level guard restricts preview use to
clients that send the `Ate-Preview-Allowed` header.

## Interface

- `preview.go:Gate` - a gate name; known gates `preview.go:GateExternalVolumes`, `preview.go:GateGoldenEgressPolicy`, `preview.go:GatePerTemplateSnapshotLocation`; `preview.go:All` (`"ALL"`) enables every gate.
- `preview.go:Init` - sets the process-wide gate state from `--preview` values; panics on a second call; unknown names are errors listing supported gates (`preview.go:newGateMap`).
- `preview.go:Verify` - checks values without changing state.
- `preview.go:IsEnabled` - gate state; unknown gates read disabled.
- `preview.go:AsMap`/`DisabledMap` - the gate maps; callers must not modify them.
- `request.go:Header` - `Ate-Preview-Allowed` gRPC metadata key; `request.go:UnaryServerInterceptor`/`StreamServerInterceptor` mark requests that did not send it as opted out (`request.go:optOutIfNotOptIn`).
- `request.go:ContextWithOptOut`/`ContextWithOptIn`/`IsOptedOut`/`IsEnabledFor` - context-level opt-in/out; opt-in overrides opt-out for system-internal call paths.
- `request.go:DialOptionsWithOptIn` - client dial options sending the header on every call.
- `testing.go:InitForTest`/`InitForTestFake` - test-only gate initialization restored on cleanup; tests using them must not run in parallel (`testing.go:testingTB`).

## Invariants

- `Init` is called at most once per process; a second call panics (`preview.go:Init`).
- Gates are disabled by default (`preview.go:newGateMap`).
- The header guard is a coarse guard against accidental use, not access control; it applies to every enabled gate at once (`request.go:Header` doc).
- A request that did not send the header is opted out for that request's context, including streams (`request.go:UnaryServerInterceptor`, `request.go:restrictedStream`).
- `InitForTest` sets `ATE_PREVIEW_SET_FOR_TEST=1` via `tb.Setenv`, which fails parallel tests (`testing.go:InitForTestFake`).

## Side Effects

- Mutates package-global state under `preview.go:globalInitLock` (`preview.go:Init`, `testing.go:InitForTestFake`).
- Logs the resulting gate state at init (`preview.go:Init`).

## Verification

- Run: `go test ./internal/preview/...`
