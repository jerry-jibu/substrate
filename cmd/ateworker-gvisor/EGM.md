# ateworker-gvisor

## Overview
The gVisor ateworker binary: hosts actor sandboxes under runsc on a worker Pod. `main.go:AteWorkerService` implements the `ateworker.Worker` gRPC service (`internal/proto/ateworkerpb`), running each actor's containers through runsc, checkpointing and restoring them, and reporting usage. `main_unsupported.go` builds on unsupported platforms.

## Interface
- `main.go:AteWorkerService`, built with `main.go:NewService(tunnel *atunnel.Tunnel, actorLogger *actorlog.ActorLogger, maxActors int, flags *flagStore, bootID string)`, serves the ateworkerpb RPCs.
- RPCs: `main.go:RunWorkload`, `main.go:CheckpointWorkload`, `main.go:RestoreWorkload`, `main.go:TerminateWorkload`, `stats.go:GetWorkloadStats`, `stats.go:GetActiveWorkloadStats`.
- Hosted actor state: `hosted.go:hostedActor` (attribution, network session, workload session, usage, memory limit, sandbox pin, memory guard); admission via `hosted.go:admitActor`.
- Container runtime: `runsc.go` wraps the runsc command; paths in `actorpath.go` (`ociBundlePath`, `runscStateDir`, `pidFilePath`, `resetRunscStateAndPidFileDirs`).
- Checkpointing: `fscheckpoint.go` (rootfs and durable snapshot file lists, `fsRestoreArgs`), `ckptcgroup.go` (`isolateCheckpointCache`, `checkpointIsolated`, `dropActorCheckpointCacheAsync`), `sandboxexit.go` (sandbox process pinning across MEMORY checkpoints), `crash.go` (crash reporting hook).
- Flags: `flags.go:flagStore` watches the flag config file and reloads; `flags.go:newFlagStore`, `parseFlagConfig`.
- Stats and limits: `stats.go` reads sandbox cgroups (`defaultCgroupRoot`, `sandboxCgroupContainer`), `memlimit.go` (`swapMemoryMaxForHigh`), `memory.go` (`unbookedMemory`), `rootfs.go:setupAppRootfs`, `phaselog.go` phase timing records.
- Children: `./internal/cdiinject/EGM.md`, `./internal/cgroupstats/EGM.md`.

## Invariants
- `admitActor` reserves capacity before network setup; a re-admitted actor cancels its previous activation and the old sandbox exit stops being a crash (`hosted.go:admitActor`).
- `sandboxProcess` pins the runsc sandbox PID so nothing waiting on it later is fooled by a reused pid (`hosted.go:hostedActor.sandbox` doc).
- `main.go:validateRunscPath` and `main.go:validateFidelity` reject malformed runsc paths and snapshot fidelity values before work starts.
- `main.go:minUsageSampleInterval` floors `--usage-sample-interval`; `main.go:workloadGracePeriod` bounds the whole shutdown drain.
- Checkpoint cache isolation moves the sandbox into a dedicated checkpoint cgroup so a checkpoint cannot evict the worker pod (`ckptcgroup.go:isolateCheckpointCache`).

## Side Effects
- Creates and deletes runsc sandboxes, OCI bundles, pid files, and per-actor directories; `resetRunscStateAndPidFileDirs` scrubs actor dirs across lifecycle transitions (`actorpath.go`).
- Writes and reads cgroup files for checkpoint isolation and memory limits (`ckptcgroup.go:writeCgroupPID`, `memlimit.go`).
- Serves the ateworker gRPC service over the atunnel network and exports usage stats.

## Verification
- Run: `go test ./cmd/ateworker-gvisor/...`
