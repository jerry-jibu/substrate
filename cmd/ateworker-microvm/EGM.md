# ateworker-microvm

## Overview

The cloud-hypervisor microVM implementation of the `ateworkerpb.Worker`
gRPC service, a peer to `cmd/ateworker-gvisor`. It runs each actor as a
cloud-hypervisor micro-VM booted via the kata guest model (kata-agent as
guest userspace, virtio-fs shared rootfs), with full suspend/resume driven
through CH's native snapshot/restore. Linux-only (`main_unsupported.go`
exits on other platforms). The VMM support packages live under
[./internal/EGM.md](./internal/EGM.md) (ch, kata, child, agentstats).

## Interface

- `main.go:main` - flags: `--pod-uid`, `--pod-ips`, `--cloud-hypervisor-binary`, `--guest-debug`, `--otlp-relay-socket`, atunnel flags (`atunnel.RegisterFlags`), `--readiness-listen-address` (`:8080`), `--max-actors` (1000), `--usage-sample-interval` (floor `main.go:minUsageSampleInterval` 50s), `--version`, `--log-level`, `--preview`.
- `main.go:AteWorkerService` - implements `ateworkerpb.WorkerServer`: `run.go:RunWorkload`, `restore.go:RestoreWorkload`, `checkpoint.go:CheckpointWorkload`, `checkpoint.go:TerminateWorkload`, `stats.go:GetWorkloadStats`, `stats.go:GetActiveWorkloadStats`.
- `main.go:beginRPC`/`rejectIfDraining`/`cancelStartups` - in-flight RPC tracking and drain rejection (`codes.Unavailable` once `shuttingDown` is set).
- `run.go:coldBootActorRetrying` - cold boot retried on `run.go:errGuestStopped` up to `run.go:coldBootAttempts` (2).
- `run.go:buildActorContainers`, `run.go:stageMergedRootfs`, `run.go:startActorContainers` - OCI spec build, host-merged overlay rootfs staging under one virtiofsd, and kata-agent sandbox/network/container setup.
- `run.go:buildVMConfig` - CH VmConfig: virtio-pmem guest image (DAX), one virtio-fs device, virtio-console `hvc0`, vsock cid 3, balloon; `run.go:agentInit` picks kata-agent-as-PID-1 only when the VMM advances guest clock on restore.
- `run.go:resolveGuestMemMiB` - guest RAM equals the actor's memory limit; rejects limits below `run.go:minGuestMemMiB` (128 MiB).
- `run.go:dialAgentRetry` - fast vsock agent dial poll; ENOENT maps to `errGuestStopped`.
- `restore.go:restoreMemoryFidelity` - MEMORY restore: `restore.go:rewriteSnapshotPaths`, background upper untar, tap rebuild with fresh FDs, `ch.RestoreWithNetFDs` in OnDemand or Eager mode (`restore.go:restoreMemMode`), CRNG reseed (`restore.go:reseedGuestCRNG`) before egress activates.
- `restore.go:restoreGuestReadyError` - classifies resume/agent-dial failures as retryable Unavailable.
- `checkpoint.go:snapshotVMState` - writes `run.go:baseIDFile` and the CH snapshot; `checkpoint.go:mergeOnDemandDelta` overlays the sparse delta onto the restore source unless eager (`snapshotIsSelfContained`).
- `checkpoint.go:writeSnapshot`/`writeBackSnapshot` - snapshot write under `checkpoint.go:snapshotWriteSlots` (2) with memborrow accounting and cgroup memory.high handling.
- `checkpoint.go:teardownActor` - stops CH then virtiofsd, closes the agent client, sweeps sandbox state, removes the rootfs upper dir, unmounts bundle overlays.
- `shutdown.go:gracefulShutdown` - SIGTERM handler: sets `shuttingDown`, cancels startups, waits in-flight RPCs (checkpoints allowed mid-drain), signals every guest workload SIGTERM then SIGKILL within `shutdown.go:workloadGracePeriod` (30m).
- `hosted.go:admitActor`/`hostActor`/`unhostActor` - capacity bounded by `--max-actors` (`ResourceExhausted` when full, draining slots included), per-actor `actornet.SandboxSession`.
- `crash.go:watchVM` - detects VMM/virtiofsd death, tears the actor down, reports via `crash.go:crashReporter` (`register.CrashReporter.ReportCrashed`) within `crash.go:crashReportTimeout` (5m).
- `stats.go:sampleGuest`/`sumContainerStats` - guest-side stats via kata-agent `StatsContainer` per workload container, bounded by `stats.go:statsCallTimeout` (2s); `stats.go:sweepUsage`/`recordInitial`/`readFinal`/`recordFinal` write usage records.
- `memlimit.go:leafMemoryHigh`/`watchMemory`/`checkMemory` - memory.high at the actor's booking (`snapshotborrow.go:actorBooking`) when PSI is available; kills actors stuck at it.
- `snapshotborrow.go:unbookedMemory`/`archiveDir` - memory ledger feeding capacity reporting and tar creation with write-behind control.
- `durable.go:tarDurableVolumes`/`untarDurableVolumes` - one `durable-dir-<volume>.tar` per durable volume; `csi.go:stageCsiVolumes`; `systeminfo.go:stageSystemInfoVolumes` (read-only remount).
- `rootfsupper.go:stageRootfsUpper`/`tarRootfsUpper`/`untarRootfsUpper` - `rootfs-upper-<container>.tar` per container upperdir.
- `fidelity.go:includesRootfs`/`includesMemory` - the fidelity ladder (VOLUMES < ROOTFS < MEMORY); `main.go:supportedFidelities` serves all three.
- `net.go:setupActorTap` - tap with fixed `net.go:gatewayMAC`/`net.go:actorGuestMAC`, MTU `net.go:actorTapMTU` (1500).
- `snapshotcache.go:dropStagedMemoryImage`/`dropActorSnapshotCacheAsync` - page-cache eviction of staged snapshot data.
- `actorpath.go:rootfsUpperDir` - worker-local dir `<root_dir>/rootfs-upper`; atelet never touches it.

## Invariants

- The kernel/image runtime assets must be staged by atelet and named in `runtime_asset_paths`; the request's `sandbox_assets_dir` must match the one derived from the actor UID (`main.go:validateRuntimeAssetPaths`), so a request cannot name its own asset root.
- The rootfs RO lower never ships in a snapshot; it is reconstructed from the OCI image at restore, and only uppers and durable volumes ship as tars (`rootfsupper.go` and `durable.go` package comments).
- System-info volumes are never captured into snapshots and are enforced read-only twice (host remount plus per-container `ro`) (`systeminfo.go` package comment, `systeminfo.go:stageSystemInfoVolumes`).
- Tars are coherent because the guest is paused first and virtiofsd serves the share write-through (`checkpoint.go:CheckpointWorkload` doc, `durable.go` comment).
- The base sandbox id is frozen across a restore lineage and propagated via `run.go:baseIDFile`; it cannot be derived from `config.json` (`run.go:baseIDFile` doc).
- Gateway and guest MACs are fixed so a restored guest's frozen ARP/interface config stays valid (`net.go:gatewayMAC`, `net.go:actorGuestMAC`).
- Guest CRNG is reseeded after every MEMORY restore before egress opens; a failed reseed fails the restore (`restore.go:reseedGuestCRNG`).
- Every restored guest cloned from one snapshot would otherwise share CRNG state; the reseed is the mitigation and has a documented post-Resume race window (`restore.go` reseed comment).
- GetWorkloadStats never takes the actor lifecycle lock and maps "no numbers" to FailedPrecondition, NOT_FOUND only when the actor is not hosted (`stats.go:GetWorkloadStats`).
- Stale guest targets (target UID mismatching attribution) are bugs: Internal error, sweep logs (`stats.go:errStaleGuestTarget`).
- Graceful shutdown keeps serving: new workloads get Unavailable, a suspend arriving mid-drain is honored (`main.go` SIGTERM comment, `shutdown.go:gracefulShutdown`).
- Signal delivery is bounded by `shutdown.go:signalDeliveryTimeout` (10s), separate from the actor's grace period.
- Snapshot config rewrites use temp-file-plus-rename, never `O_TRUNC`, because the file may share an inode (`restore.go:rewriteSnapshotPaths`).
- A cold boot starts from a pristine upper dir; any previous activation's contents are wiped by ateworker, not atelet (`rootfsupper.go:resetRootfsUpperDir`).
- Capacity stays occupied while network cleanup drains (`hosted.go:unhostActor` `draining` counter).

## Side Effects

- Mounts: self-binds `/run/kata-containers` rshared for kata virtio-fs propagation (`main.go:ensureSharedPropagation`); builds taps in the actor's netns (`net.go:setupActorTap`); binds durable/CSI/system-info dirs into the virtio-fs share (`durable.go:stageDurableVolumes`, `csi.go:stageCsiVolumes`, `systeminfo.go:stageSystemInfoVolumes`); composes bundle rootfs overlays (`imagecache.SetupBundleRootfs` via `run.go:buildActorContainers`).
- Processes: launches and owns cloud-hypervisor and virtiofsd per actor (`run.go:coldBootActor`, `checkpoint.go:teardownActor`); delegates a worker cgroup (`cgroup.Delegate`).
- Writes: worker socket at `nodepath.WorkerSocketPath` (stale socket removed at boot), checkpoint dir files (CH snapshot + tars + base-id), rootfs upper tars, snapshot `config.json` rewrite on restore.
- Network: drives `atunnel` egress/ingress per actor (`tunnel.Deactivate`/`PrepareEgress`/`ActivateEgress`/`ActivateIngress`); telemetry exported via `otlprelay.Dial` with fallback to `OTEL_EXPORTER_OTLP_ENDPOINT`.
- gRPC: serves `ateworkerpb.Worker` on a unix socket with OTel stats handler and `ateinterceptors.InternalServerUnaryInterceptor`; health/readiness on `--readiness-listen-address`; registers capacity via `register.Report` and epochs (`register.Epoch.Require` gates Run/Restore).
- Object storage: none directly; atelet uploads the checkpoint dir after CheckpointWorkload returns (`checkpoint.go` contract comment).

## Verification

- Run: `go test ./cmd/ateworker-microvm/...`
