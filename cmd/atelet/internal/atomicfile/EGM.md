# atomicfile

## Overview

Writes files that readers never observe partly written, via a temp file in the
target directory plus rename.

## Interface

- `atomicfile.go:Write` - writes `data` to `path` atomically (temp file + rename); does not fsync.
- `atomicfile.go:WriteSync` - `Write` for a file that must survive a node crash.

## Invariants

- Readers never observe a truncated or partially written value: the temp file `.path.tmp-*` is renamed over the target only after a successful write (`atomicfile.go:write`).
- After `WriteSync` returns, the file is durable with exactly the written content: data is synced before the rename and the parent directory after it (`atomicfile.go:WriteSync`).
- `Write` does not fsync, so after a node crash the path can be empty or still hold its old contents (`atomicfile.go:Write`).

## Side Effects

- Filesystem: creates a temp file in the target's directory, renames it over the target, and removes the temp file on failure (`atomicfile.go:write`).
- `WriteSync` issues `fsync` on the file and on the parent directory.

## Verification

- Run: `go test ./cmd/atelet/internal/atomicfile/...`
