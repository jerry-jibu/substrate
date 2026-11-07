# actordirs

## Overview

Manages each actor's on-node directory tree for atelet: emptying it between
activations and reclaiming it at termination. Path layout itself comes from
`cmd/atelet/internal/ateletpath`; this package performs the removals and
recreations, using `internal/imagecache.RemoveAllWritable` where copied-up
image directories keep read-only modes.

## Interface

- `actordirs.go:ResetDir` - empties `dir` via a caller-supplied remove func and recreates it with `mode`.
- `actordirs.go:ResetActorDirs` - empties the actor's directories and leaves them in place for the next activation.
- `actordirs.go:ResetActorDirsKeepingVolumes` - reset variant for Run/Restore that leaves external volume mount points alone.
- `actordirs.go:RemoveActorFilesExceptState` - removes the actor's snapshot tree and all files under the actor dir except `snapshot-state.json`.
- `actordirs.go:RemoveActorDirs` - reclaims both of the actor's directory trees, roots included.
- `actordirs.go:removeEmptyVolumeDirs` (unexported) - removes empty mountpoint directories under the external volumes dir.

## Invariants

- Refuses to delete through a populated external volume mount directory, so a failed unmount cannot turn into a deletion of the mount's contents (`actordirs.go:removeEmptyVolumeDirs`, `actordirs.go:RemoveActorDirs`).
- The OCI bundle dir is removed with `imagecache.RemoveAllWritable`, not `os.RemoveAll`, because the bundle's upper dir can carry the image's read-only modes (`actordirs.go:ResetActorDirsKeepingVolumes`).
- `ResetActorDirsKeepingVolumes` leaves the runsc logs dir and the external volume mount points untouched; republishing a volume left by a failed attempt is a no-op (`actordirs.go:ResetActorDirsKeepingVolumes`).
- System-info volume roots are recreated world-readable (`0o755`) because they are bind-mounted read-only into the actor (`actordirs.go:ResetActorDirsKeepingVolumes`).
- Terminate deletes `snapshot-state.json` (Forget) before `RemoveActorDirs` runs; after a crash mid-cleanup only a retried Terminate finishes it, because Start finds no state file (`actordirs.go:RemoveActorDirs`).

## Side Effects

- Deletes and recreates directories under `ateletpath.ActorPath` and `ateletpath.ActorSnapshotPath` on the node filesystem (bundle, checkpoint-state, restore-state, durable-dir mounts, system-info roots, volumes).
- Removes the actor's snapshot staging tree and the actor directory roots themselves on termination.

## Verification

- Run: `go test ./cmd/atelet/internal/actordirs/...`
