# ch

## Overview
Cloud Hypervisor (CLH) support for the microVM worker: building VM configs, driving the CLH HTTP API over its Unix socket, merging sparse rootfs overlays for restore, launching the VMM with FD passing, prefaulting memory, and restoring guest clocks.

## Interface
- `ch.go:Client`, built with `ch.go:NewClient(apiSocket string)`, speaks to the CLH API socket; `ch.go:VMMInfo` describes a running VMM. `ch.go:SnapshotURL(dir)` names the snapshot location.
- VM config types in `createvm.go`: `VmConfig` and its sections (`BalloonConfig`, `FsConfig`, `PlatformConfig`, `CpusConfig`, `MemoryConfig`, `PayloadConfig`, `PmemConfig`, `RngConfig`, `ConsoleConfig`, `VsockConfig`).
- Launch and restore: `restorefds.go:LaunchVMM(ctx, LaunchVMMOptions)` starts the VMM child and returns `(*child.Process, *Client)`; `restorefds.go:SnapshotNetDevices(snapshotDir)` recovers network device FDs; `restorefds.go:RestoredNet` carries restored network state.
- Overlay merge: `merge.go:MergeSparseOverlay(ctx, baseFile, deltaFile, outFile)` and `merge.go:MergeDeltaIntoBase(ctx, baseFile, deltaFile)`.
- Support: `prefault.go` (memory prefaulting), `guestclock.go` (guest clock restoration), `restorefds.go` file (`api.go` holds the API request surface).

## Invariants
- Snapshot location is always a `file://` URL over the snapshot dir (`ch.go:SnapshotURL`).
- The VMM runs as a supervised `child.Process`; its lifecycle is owned by the caller of `LaunchVMM`.

## Side Effects
- Talks to the cloud-hypervisor HTTP API over the VM's Unix socket (`ch.go:Client`).
- Reads and writes sparse overlay and snapshot files on disk (`merge.go`, `restorefds.go:SnapshotNetDevices`).
- Launches OS processes (the VMM) with inherited file descriptors (`restorefds.go:LaunchVMM`).

## Verification
- Run: `go test ./cmd/ateworker-microvm/internal/ch/...` (Linux-specific behavior is build-tagged; tests skip off-Linux).
