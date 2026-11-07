# freemap

## Overview

Fixed-capacity free-slot tracker for 64k entries, implemented as a three-level
hierarchical bit-set (~8 KiB). The DNS relay uses it to allocate and recycle
16-bit transaction IDs without collisions.

## Interface

- `Map64k` (`freemap64k.go:Map64k`): the free map type; zero value is an empty map ready to use.
- `(*Map64k).Count() int` (`freemap64k.go:Count`): number of bits currently set.
- `(*Map64k).IsFull() bool` (`freemap64k.go:IsFull`): true when all 65536 bits are set.
- `(*Map64k).Get(index uint16) bool` (`freemap64k.go:Get`): reports whether an index is occupied.
- `(*Map64k).Set(index uint16)` (`freemap64k.go:Set`): marks an index occupied.
- `(*Map64k).Clear(index uint16)` (`freemap64k.go:Clear`): marks an index free.
- `(*Map64k).FindFirstUnset() (uint16, bool)` (`freemap64k.go:FindFirstUnset`): lowest free index, false if full.
- `(*Map64k).FindFirstUnsetFrom(index uint16) (uint16, bool)` (`freemap64k.go:FindFirstUnsetFrom`): first free index at or after `index`, wrapping around the ID space.

## Invariants

- Capacity is exactly 65536 entries (`freemap64k.go:map64kBits`).
- `Set` on an already-occupied index and `Clear` on a free index are no-ops; `count` stays consistent with the number of set bits (`freemap64k.go:Set`, `freemap64k.go:Clear`).
- Layer-1/layer-2 summary bits are maintained only while a lower word is completely full, and propagate on the transition (`freemap64k.go:Set`, `freemap64k.go:Clear`).
- Not thread-safe; callers must synchronize (`freemap64k.go:Map64k` doc comment).

## Side Effects

- None. Pure in-memory data structure.

## Verification

- Run: `go test ./internal/atunnel/dns/freemap/...`
