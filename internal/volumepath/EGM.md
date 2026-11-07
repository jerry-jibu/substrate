# volumepath

## Overview

The rule for the relative paths at which substrate projects files into an
actor's volumes. ateapi applies it when an ActorTemplate is created and
atelet applies it again before writing to the host filesystem; both call
this one implementation so the two cannot drift.

## Interface

- `volumepath.go:ValidateProjected` - validates a projected file path relative to the volume root; `volumepath.go:MaxSegments` (16) bounds depth.
- Subdirectories: none.

## Invariants

- A path must be non-empty, NUL-free, at most 16 `/`-separated segments, with no empty, `.`, or `..` segment (strictly relative) (`volumepath.go:ValidateProjected`).
- Any other byte is allowed: Linux paths are bytes, so Unicode in any encoding passes (`volumepath.go:ValidateProjected` doc).

## Side Effects

- ["None"]

## Verification

- Run: `go test ./internal/volumepath/...`
