# gcsboundary

## Overview
Turns actor grants into the Credential Access Boundary rules a downscoped GCS token carries ("Package gcsboundary", `boundary.go`). One rule per grant, each limited to its bucket and an object-name prefix.

## Interface
- `boundary.go:FromGrants(grants []actorscope.Grant) ([]downscope.AccessBoundaryRule, error)` builds one access boundary rule per grant.

## Invariants
- Each rule is scoped to exactly one bucket and one object-name prefix, taken from the grant ("Package gcsboundary" doc).

## Side Effects
- ["None"] (pure mapping; the downscoping happens in `gcsissuer`).

## Verification
- Run: `go test ./internal/objectstoreplugin/gcsboundary/...`
