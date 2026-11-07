# listers

## Overview
Root of the generated lister tree (`lister-gen` output) for the Substrate CRD types. Contains one group directory, `ate`.

## Subdirectories & Boundaries
- ate: [./ate/EGM.md](./ate/EGM.md) - group directory for the `ate` API listers, whose `v1` package holds the `CSIDriverConfig`, `SandboxConfig`, and `WorkerPool` listers.

## Invariants
- None (no direct source files).

## Side Effects
- None.

## Verification
- Run: `go test ./pkg/client/listers/...` from the repo root.
