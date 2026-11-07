# bootid

## Overview
Reads the Linux boot ID, a random UUID that changes on every reboot and on nothing else, used to key node-scoped state to a boot.

## Interface
- `bootid.go:Path` - `/proc/sys/kernel/random/boot_id`, where Linux exposes the boot ID.
- `bootid.go:Read` - reads and returns the boot ID from a path.

## Invariants
- The value is never empty, so a record that has none never matches it (`bootid.go:Read`).
- The ID is not namespaced: a pod reads the node's (`bootid.go:Path`).

## Side Effects
- Reads from the filesystem (`/proc`) at each call (`bootid.go:Read`).

## Verification
- Run: `go test ./internal/bootid/...`
