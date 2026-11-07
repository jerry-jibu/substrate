# actorscope

## Overview

Derives the object-storage prefixes an actor's transfers may touch, with the access each needs, from the actor record. Knows no storage provider: a provider's credential exchange turns the grants into whatever scoped credential its storage supports.

## Interface

- `actorscope.go:FromActor(actor *ateapi.Actor) ([]Grant, error)` - derives grants from `status.snapshots`.
- `actorscope.go:Grant` - `Prefix resources.StoragePrefix` plus `Access`.
- `actorscope.go:Access` - `ReadOnly` (the tag or golden snapshot the actor was created from) and `ReadWrite` (the actor's own snapshots), stringified "ro"/"rw".

## Invariants

- Every durable snapshot URI grants ReadWrite on the actor's own prefix under its location, `<location>/atespaces/<atespace>/actors/<uid>`, from the first generation on; a repointed actor keeps both locations (`actorscope.go:FromActor`).
- Only a TAG-owned generation in `SNAPSHOT_STORAGE_STATE_COMPLETED` grants ReadOnly on the tag's prefix (`actorscope.go:FromActor`).
- Prefixes are parsed and built with the same code (`resources.ParseSnapshotURI`, `resources.ActorSnapshotOwner`) the control plane used to write them (`actorscope.go:FromActor`).
- An unknown snapshot owner is an error, so a new kind is never silently dropped; local-only generations contribute nothing (`actorscope.go:FromActor`).
- Grants are sorted ReadWrite-first then by prefix, so equal inputs give equal output (`actorscope.go:FromActor`).

## Side Effects

- ["None"] (pure derivation over the actor record).

## Verification

- Run: `go test ./internal/objectstoreplugin/actorscope/...`
