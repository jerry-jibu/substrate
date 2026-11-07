# objectstore

## Overview
Public packages for out-of-tree object-store providers. `actorauthz` is the credential-exchange authorization library such a provider embeds.

## Subdirectories & Boundaries
- actorauthz: [./actorauthz/EGM.md](./actorauthz/EGM.md) - verifies worker-for-actor JWTs and returns the storage prefixes (`Grant`s) a provider-issued credential must be limited to.

## Invariants
- None (no direct source files).

## Side Effects
- None.

## Verification
- Run: `go test ./pkg/objectstore/...` from the repo root.
