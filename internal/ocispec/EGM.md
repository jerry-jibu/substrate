# ocispec

## Overview

Builds the runtime-neutral OCI spec for actor bundles, which each worker
shapes for its runtime: `microvm.go` shapes for kata/cloud-hypervisor,
`gvisor.go` for runsc.

## Interface

- `ocispec.go:Build` - runtime-neutral spec for one actor container: process (user, args, env, cwd, capabilities, `RLIMIT_NOFILE` 1048576), rootfs `rootfs`, `ocispec.go:systemMounts`, namespaces from `ocispec.go:namespaces` (network namespace at `nodepath.ActorNetNSPath`), resources via `ocispec.go:ociResources`, and volume binds (durable rw, external/CSI rw, system-info ro, image volumes ro, all `nodev`).
- `ocispec.go:BuildSandbox` - spec of a sandbox with no root container (hostname and namespaces only).
- `ocispec.go:Save` - writes the spec as `config.json` in the bundle, mode 0600.
- `ocispec.go:Hostname` - `actor`, the UTS hostname for actor containers.
- `microvm.go:ShapeMicroVM` - repoints volume binds to guest share paths (`microvm.go:guestVolumeSource`), `bind` becomes `rbind`, masks containerd's default dirs/files that exist in the guest, sets `net.ipv4.ip_unprivileged_port_start=0` and `net.ipv4.ping_group_range`, merges kata default resources (`microvm.go:mergeKataResources`, `microvm.go:guestResources`).
- `microvm.go:GuestContainerID` - `c_<name>`, the kata container id; `microvm.go:GuestSharedDir` and `ShareDurable`/`ShareCSI`/`ShareSystemInfo`/`ShareVolumes` name the sub-share layout.
- `gvisor.go:ShapeGVisor` - adds runsc CRI annotations and durable-volume mount hints, drops mounts runsc replaces, caps `/tmp` at half the memory limit, sets a default colon-free cgroupsPath (`gvisor.go:GVisorCgroupLeaf`), applies `sizing.SandboxSize.ApplyToOCISpec`.
- `gvisor.go:SandboxID` - `_sandbox`, the runsc sandbox id with `-no-root-container`.

## Invariants

- `GuestContainerID` depends only on the container name, because a restored guest keeps the ids it booted with; `_` cannot appear in a k8s name, so ids cannot collide with names or `Share*` dirs (`microvm.go:GuestContainerID` doc).
- `ShapeMicroVM` must run on an unshaped spec and errors on a bind it cannot place in the guest (`microvm.go:ShapeMicroVM` doc).
- `mergeKataResources` fills gaps rather than allowlisting, so upstream resource fields reach the guest (`microvm.go:mergeKataResources` doc).
- `ShapeGVisor` is idempotent; runsc requires unchanged mount options across MEMORY restores, hence the fixed `/tmp` size (`gvisor.go:ShapeGVisor` doc).
- A volume at a system mount path (e.g. `/tmp`) replaces that mount in place; nesting mounts are rejected upstream (`ocispec.go:Build` comment).
- Capabilities inherit via Bounding only; Inheritable and Ambient stay empty (gVisor limitation, `ocispec.go:Build` TODO).
- CPU quota uses `ocispec.go:cpuQuotaPeriodUS` (100000) with a floor of `cpuQuotaMinUS` (1000 microseconds), so limits under 10m are raised instead of producing a spec the guest refuses (`ocispec.go:ociResources`).

## Side Effects

- `Save` writes `config.json` to disk (0600) (`ocispec.go:Save`).

## Verification

- Run: `go test ./internal/ocispec/...`
