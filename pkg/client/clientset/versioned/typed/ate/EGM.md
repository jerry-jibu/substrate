# clientset/versioned/typed/ate

## Overview
Group directory for the typed clients of the `ate` API group. Only version `v1` exists.

## Subdirectories & Boundaries
- v1: [./v1/EGM.md](./v1/EGM.md) - typed `AteV1Client` with CRUD interfaces for `CSIDriverConfig`, `SandboxConfig`, and `WorkerPool`, plus its generated fake under `v1/fake`.

## Invariants
- None (no direct source files).

## Side Effects
- None.

## Verification
- Run: `go test ./pkg/client/clientset/versioned/typed/ate/...` from the repo root.
