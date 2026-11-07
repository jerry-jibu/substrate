# listers/ate

## Overview
Group directory holding the generated listers for the `ate` API group. Only version `v1` exists.

## Subdirectories & Boundaries
- v1: [./v1/EGM.md](./v1/EGM.md) - generated listers (`CSIDriverConfigLister`, `SandboxConfigLister`, `WorkerPoolLister`) that read from an informer's indexer instead of the API server.

## Invariants
- None (no direct source files).

## Side Effects
- None.

## Verification
- Run: `go test ./pkg/client/listers/ate/...` from the repo root.
