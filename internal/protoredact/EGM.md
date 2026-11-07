# protoredact

## Overview

Masks the fields of a protobuf message marked `[debug_redact = true]` in
the schema, so the message can be logged without exposing its secrets. The
Go protobuf runtime does not act on the option itself; this package is its
reader.

## Interface

- `protoredact.go:NeedsRedaction` - true when m is a valid message whose type can reach a labeled field and at least one is populated; the schema check is memoized (`protoredact.go:reachesDebugRedact`, `protoredact.go:reachable`).
- `protoredact.go:Redacted` - returns m unchanged plus false when nothing needs masking, else a masked deep copy plus true.
- `protoredact.go:Placeholder` - `[REDACTED]`, replacing singular string fields; other kinds are cleared (`protoredact.go:redact`).
- `protoredact.go:hasDebugRedact` - reads `[debug_redact = true]` from `descriptorpb.FieldOptions`.
- Subdirectories: none.

## Invariants

- `Redacted` never modifies m; the no-mask path returns m itself and callers must treat it as read-only, and must use the bool rather than comparing messages (`protoredact.go:Redacted` doc).
- A type that admits extension fields counts as reaching a secret, and the populated walk covers extensions via Range, since `Fields()` does not list them (`protoredact.go:walkReachesDebugRedact`, `protoredact.go:redact`).
- Recursive schemas are guarded by a seen set (`protoredact.go:walkReachesDebugRedact`).
- Only singular string fields get the placeholder; bytes, repeated, map, and message fields are cleared because no honest placeholder exists (`protoredact.go:redact` doc).
- Sub-messages whose type cannot carry a labeled field are never visited (`protoredact.go:redact`).

## Side Effects

- ["None"] (pure reflection over message values; the clone in Redacted is a new value)

## Verification

- Run: `go test ./internal/protoredact/...`
