# ateapi/internal/defaults

## Overview

Applies each resource's field defaults in place. It is its own package, separate
from validation, so that both the RPC handlers and the store layer (which
backfills defaults on read, without depending on the RPC handlers) can call it
(`defaults.go` package doc).

## Interface

- `Apply(m proto.Message)` (`defaults.go`) - defaults an AccessPolicy, Actor, ActorTemplate, Atespace, EgressPolicy, Tag, or Worker in place; panics on any other message type.
- Per-type defaulting, e.g. `applyActorTemplateDefaults`, `applySnapshotConfigDefaults`, `applyContainerDefaults` (`defaults.go`).

## Invariants

- An omitted `snapshot_config` takes every default; unspecified fidelity becomes `SNAPSHOT_FIDELITY_MEMORY` and unspecified golden mode becomes `GOLDEN_SNAPSHOT_MODE_ENABLED` (`defaults.go` `applySnapshotConfigDefaults`).
- `Apply` panics on an unknown resource message rather than silently passing it through (`defaults.go` `Apply`).

## Side Effects

- Mutates the passed proto message in place (`defaults.go`).

## Verification

- Run: `go test ./cmd/ateapi/internal/defaults/...`
