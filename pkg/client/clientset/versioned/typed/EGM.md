# clientset/versioned/typed

## Overview
Groups the generated typed clients by API group. Contains the single group directory `ate`.

## Subdirectories & Boundaries
- ate: [./ate/EGM.md](./ate/EGM.md) - typed REST clients and fakes for the `ate` group, version `v1`.

## Invariants
- None (no direct source files).

## Side Effects
- None.

## Verification
- Run: `go test ./pkg/client/clientset/versioned/typed/...` from the repo root.
