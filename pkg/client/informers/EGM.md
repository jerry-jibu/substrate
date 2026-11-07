# informers

## Overview
Root of the generated informer tree (`informer-gen` output). Contains the `externalversions` factory tree for Substrate's CRD types.

## Subdirectories & Boundaries
- externalversions: [./externalversions/EGM.md](./externalversions/EGM.md) - the `SharedInformerFactory` that caches one informer per `ate/v1` resource type and exposes group, version, and generic (`ForResource`) access.

## Invariants
- None (no direct source files).

## Side Effects
- None.

## Verification
- Run: `go test ./pkg/client/informers/...` from the repo root.
