# clientset

## Overview
Root of the generated clientset tree (`client-gen` output). The concrete package is `versioned`; this directory only groups it.

## Subdirectories & Boundaries
- versioned: [./versioned/EGM.md](./versioned/EGM.md) - the versioned `Clientset` (`Interface`, `NewForConfig`), its `typed/ate/v1` clients and fakes, and its `scheme` package.

## Invariants
- None (no direct source files).

## Side Effects
- None.

## Verification
- Run: `go test ./pkg/client/clientset/...` from the repo root.
