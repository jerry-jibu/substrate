# ateletpath

## Overview

Atelet's on-node path layout: the per-actor directories it passes to worker as
`ActorDirs`, and the node-local directories only atelet uses. Pure path
computation over `internal/nodepath` constants; no filesystem access.

## Interface

- `ateletpath.go:ImageCacheDir`, `SnapshotScratchDir`, `SandboxAssetCacheDir`, `SandboxAssetScratchDir`, `SandboxRuntimeIndexFile`, `CacheBootIDFile`, `SnapshotCacheDir` - node-level path variables under `nodepath.BasePath`.
- `ateletpath.go:ActorPath`, `ActorSnapshotPath` - the actor's two directory roots (actor dir and snapshot staging dir).
- `ateletpath.go:ActorSandboxAssetsFile`, `ActorSnapshotStateFile` - per-actor state files that survive directory resets.
- `ateletpath.go:SandboxAssetsDir`, `SandboxAssetsStagingDir`, `OCIBundleDir`, `OCIBundlePath` - activation asset and OCI bundle paths.
- `ateletpath.go:CheckpointStateDir`, `LocalCheckpointsDir`, `LocalSnapshotDir`, `RestoreStateDir` - checkpoint and restore paths under the snapshot staging dir.
- `ateletpath.go:DurableDirVolumeMountsDir`, `DurableDirVolumeMountPoint`, `SystemInfoVolumeRootsDir`, `SystemInfoVolumeRoot` - volume mount locations.
- `ateletpath.go:ExternalVolumeMountsDir`, `ExternalVolumeHostPath` - external volume paths.
- `ateletpath.go:ActorDirs` - builds the `ateworkerpb.ActorDirs` message atelet passes to worker; worker takes these paths from the request rather than deriving them.

## Invariants

- Caches (`SandboxAssetCacheDir`, `SnapshotCacheDir`) live under `nodepath.BasePath` on the same mount as the dirs they serve by hard link, because `link(2)` requires both ends on one mounted filesystem (`ateletpath.go:SandboxAssetCacheDir`, `ateletpath.go:SnapshotCacheDir`).
- `ActorSandboxAssetsFile` and `ActorSnapshotStateFile` sit directly under `ActorPath`, outside the subdirs `actordirs.ResetActorDirs` wipes, so they survive between Run and a later Checkpoint (`ateletpath.go:ActorSandboxAssetsFile`, `ateletpath.go:ActorSnapshotStateFile`).
- Checkpoints link and rename files between `ActorSnapshotPath` subdirectories, so those subdirectories must stay on one filesystem (`ateletpath.go:ActorSnapshotPath`).
- `RestoreStateDir` is a different path from `CheckpointStateDir` because `runsc restore -direct -background` demand-pages the checkpoint while executing (`ateletpath.go:RestoreStateDir`).
- `SystemInfoVolumeRootsDir` is a separate directory from `DurableDirVolumeMountsDir` because the micro-VM sandbox captures durable data by location (tars all of `DurableDirVolumeMountsDir`); system-info contents must never be captured (`ateletpath.go:SystemInfoVolumeRootsDir`).
- `SandboxAssetsStagingDir` holds staged assets so a failed fetch leaves a live activation's `SandboxAssetsDir` untouched (`ateletpath.go:SandboxAssetsStagingDir`).

## Side Effects

- None. Pure string/path computation.

## Verification

- Run: `go test ./cmd/atelet/internal/ateletpath/...`
