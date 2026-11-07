# proto

## Overview
Public protobuf packages for external consumers. Contains the generated `ateconfigpb` configuration messages.

## Subdirectories & Boundaries
- ateconfigpb: [./ateconfigpb/EGM.md](./ateconfigpb/EGM.md) - `APIConfig`/`APILogging` configuration messages for ate-api-server, with generated Go bindings.

## Invariants
- None (no direct source files).

## Side Effects
- None.

## Verification
- Run: `go test ./pkg/proto/...` from the repo root.
