# kata

## Overview
Kata-containers-derived scaffolding for the microVM worker: path layout for CLH sandboxes, kernel parameters, the vsock agent client, virtiofsd and overlay staging for shared rootfs and volumes, OCI-to-agent spec conversion, sandbox state cleanup, and restore path helpers.

## Interface
- Paths: `kata.go:CLHSocketPath(id)`, `restore.go:VMDir(id)`, `restore.go:VsockSocketPath(id)`, `restore.go:ConsoleLogPath(id)`, `restore.go:SerialLogPath(id)`, `overlay_linux.go:SharedDir(id)`, `overlay_linux.go:VirtiofsdSocketPath(id)`, `overlay_linux.go:GuestSharedRootfs(containerID)`, `overlay_linux.go:GuestSharedVolumeDir`, `overlay_linux.go:SharedVolumeDir`, `overlay_linux.go:UpperWorkDirs`.
- Agent client: `agentclient.go:DialAgent(ctx, vsockPath)` returns `agentclient.go:AgentClient` over the vsock; `agentclient.go:NewStdioReader` streams container stdio (`StreamReader`); `agentclient.go:DebugConsoleDump(ctx, vsockPath, cmd)` runs a debug console command.
- Staging: `overlay_linux.go:StartVirtiofsd(ctx, VirtiofsdOptions)` launches virtiofsd; `overlay_linux.go:StageImageVolume`, `StageMergedRootfs`, `UnmountMergedRootfs`, `BindIntoShare`, `ReconstructSharedDirFromImage`, `MkdirUpperWorkDirs`, `Unmount`, `RemountReadOnly`; `overlay_linux.go:CreateSandboxOpts` bundles sandbox creation options.
- Spec conversion: `specconv.go:SpecToAgentPB(s *specs.Spec) *agentpb.Spec`, `specconv.go:DefaultCPUPeriodUS`.
- Config and cleanup: `config.go:BaseKernelParams`, `config.go:WithAgentDebug(kernelParams)`, `cleanup_linux.go:CleanupSandboxState(ctx, id)`.

## Invariants
- Kernel params always include `cgroup_no_v1=all systemd.unified_cgroup_hierarchy=1` (`config.go:BaseKernelParams`); cgroup v2 only.
- CPU quota conversion uses the fixed 100000us period (`specconv.go:DefaultCPUPeriodUS`).
- The guest sees its rootfs and volumes only through the virtiofsd share (`GuestSharedRootfs`/`SharedVolumeDir` layout).

## Side Effects
- Dials the kata agent over a vsock Unix socket (`agentclient.go:DialAgent`).
- Mounts, bind-mounts, and unmounts overlay upper/work dirs and volume staging paths (`overlay_linux.go`).
- Launches virtiofsd and other VMM-adjacent processes (`overlay_linux.go:StartVirtiofsd`, returning `child.Process`).
- Removes sandbox state directories and sockets on cleanup (`cleanup_linux.go:CleanupSandboxState`).

## Verification
- Run: `go test ./cmd/ateworker-microvm/internal/kata/...` (Linux-specific tests skip off-Linux).
