# volume

## Overview

Volume plugin abstraction for Agent Substrate: the control-plane and
worker-plane interfaces storage backends implement, plus a host-directory mock
used by tests and e2e runs. The CSI implementation lives under `csi/`.

## Interface

- `VolumePluginControlPlane` (`plugin.go:VolumePluginControlPlane`): `DriverName`, `CreateVolume`, `DeleteVolume`, `AttachVolume`, `DetachVolume`.
- `VolumePluginWorkerPlane` (`plugin.go:VolumePluginWorkerPlane`): `MountVolume`, `UnmountVolume`.
- Request/response types: `CreateVolumeRequest`/`CreateVolumeResponse`, `AttachVolumeRequest`/`AttachVolumeResponse`, `MountVolumeRequest` (`plugin.go:CreateVolumeRequest`).
- `MockVolumePlugin` / `NewMockVolumePlugin()` (`mock.go:NewMockVolumePlugin`): test implementation of both interfaces; driver name `substrate.io/mock` (`mock.go:DriverName`).
- Subdirectory:
  - [csi](./csi/EGM.md) - CSI driver client and adapter implementing both plugin interfaces.

## Invariants

- `PublishContext` from attach must be echoed back on the node's mount calls, and is only meaningful for the node named in the request (`plugin.go:AttachVolumeResponse` doc).
- Mock control-plane methods (Create/Delete/Attach/Detach) are intentionally stateless no-ops with no bookkeeping, because multiple ate-api-server replicas cannot share in-process state; correctness is verified through the file `MountVolume` writes, not plugin queries (`mock.go:MockVolumePlugin` doc).
- Mock volume data persists only when the actor lands on the same host (`mock.go:MockVolumePlugin` doc).
- The mock does not clean up its subdirectories; test infrastructure must (`mock.go:MockVolumePlugin` doc).

## Side Effects

- `MockVolumePlugin.MountVolume` creates `<nodepath.BasePath>/mockvolumes/<volumeID>`, writes a `test.txt` into it, and symlinks the target path to it instead of bind mounting (`mock.go:MountVolume`).
- `MockVolumePlugin.UnmountVolume` removes the target symlink, tolerating absence (`mock.go:UnmountVolume`).
- All mock operations emit slog log lines (`mock.go`).

## Verification

- Run: `go test ./internal/volume/...`
