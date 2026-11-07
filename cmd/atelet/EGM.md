# atelet

## Overview

The atelet binary: the per-node daemon that runs actors on worker pods. It
serves the `ateletpb.AteletServer` lifecycle RPCs to ate-api-server, a
worker-facing `ateletpb.WorkerSupportServer` on a node-local unix socket,
and manages the node's image, sandbox-asset and snapshot caches, snapshot
tracking and background upload/detach, sandbox runtime indexing, and CSI
volume mounts. Core lifecycle logic is shared with `internal/atelet`; this
package is the wiring, the RPC surface, and the node-local machinery.

## Interface

- `main.go:main` - flag parsing, logging/tracing/metrics boot, cache and GC startup, gRPC servers, signal-driven drain.
- `main.go:Atelet`, `main.go:NewService` - the `ateletpb.AteletServer` implementation (`var _ ateletpb.AteletServer`).
- `main.go:(*Atelet).Run`, `(*Atelet).Checkpoint`, `(*Atelet).PromoteSnapshot`, `(*Atelet).Restore`, `(*Atelet).DetachActor`, `(*Atelet).Terminate` - actor lifecycle RPCs; each validates via `internal/apivalidation` before on-node work.
- `workersupport.go:workerSupportServer` - `ateletpb.WorkerSupportServer` for workers on the node: `MintActorCertificate`, `AcquireWorkerEpoch`, `RegisterWorker`, `SetWorkerSandboxRuntimes`, `RequestActorSuspend`, `GetActorEgressPolicy`, `ReportActorCrashed`.
- `main.go:newWorkerDialer` - pooled dialing to worker pods.
- `worker_sandbox_syncer.go:newWorkerSandboxSyncer` / `worker_sandbox_versions.go` - keep each worker pod's sandbox runtimes file in sync with its pool's SandboxConfig.
- `sandbox_runtime_index.go:runtimeIndex`, `newRuntimeIndex`, `openRuntimeIndex` - node view of advertised sandbox runtimes, persisted to `ateletpath.SandboxRuntimeIndexFile`.
- `sandbox_assets.go` - sandbox asset records, staging, and the per-snapshot manifest (`sandboxManifestName`).
- `snapshotcache.go`, `sandbox_asset_cache.go`, `imagegc.go` - snapshot file cache, sandbox asset cache, and image-cache GC loop, with their flag sets.
- `snapshot_transfer.go:fetchSnapshotFiles`, `pluginCallError` - snapshot file moves through the object-store plugin.
- `worker_actor_jwt.go:workerActorJWTs` - mints and caches worker-for-actor JWTs via ate-api-server.
- `statspoller.go` - actor resource utilization sampling from worker stats.
- `systeminfovolume.go:newSystemInfoVolumeRefresher` - regenerates system-info volumes per Run/Restore.
- `volumes.go:(*Atelet).mountExternalVolumes` - CSI external volume publish/unpublish.
- `deviceplugin.go:startDevicePlugins` - advertises host devices (e.g. /dev/kvm) to kubelet as extended resources.
- `bootid.go:localSnapshotBootCheck`, `bootid.go:clearCachesAfterReboot` - boot-ID based cache clearing and snapshot freshness checks.
- `metrics.go:Instruments`, `NewInstruments` - `ate.actor.restore.duration` / `ate.actor.checkpoint.duration` histograms.
- Internal packages: [./internal/EGM.md](./internal/EGM.md).

## Invariants

- Lifecycle RPCs are serialized per actor by `actorLocks` before any on-node work (`main.go:lockActor`); the snapshottracker contract assumes this serialization.
- Run validates the request and resolves the advertised runtime before `BeginStart` or any directory reset, so an invalid request changes nothing (`main.go:(*Atelet).Run`).
- Sandbox assets are fetched into `SandboxAssetsStagingDir` and swapped in only after the actor dirs reset, so a failed fetch leaves a live activation untouched (`main.go:(*Atelet).Run`).
- Worker-facing RPC identity comes only from the mTLS certificate, never the request; the control plane mints only for the worker the actor is assigned to (`workersupport.go:MintActorCertificate`).
- Herder-facing calls are restricted to a single SPIFFE ID (`--herder-allowed-spiffe-id`, `main.go:herderAllowedSPIFFEID`); the worker support socket is chmod 0600 (`main.go`).
- `ATE_STORAGE_BACKEND` set on atelet is fatal at startup: storage settings belong to the objectstore-plugin sidecar (`main.go:rejectStorageEnv`).
- Snapshot flags are validated before caches are cleared or emptied: delays positive, upload delay <= detach delay, throughput bounded by `maxSnapshotUploadEstMiBps` (`main.go:validateSnapshotDelayFlags`).
- On a node reboot (boot-ID marker mismatch) the node-local caches are emptied and actors' bundle dirs and workers' default-runtime dirs removed (`bootid.go:clearCachesAfterReboot`).
- The runtime index map by hash never forgets an entry and is rewritten durably (atomicfile) before a new entry is added, so a runtime registered before a restart resolves at once (`sandbox_runtime_index.go:runtimeIndex`).
- Object-store plugin errors are rewrapped keeping only the gRPC code and a redacted snapshot URI, because the message may quote a signed URL (`snapshot_transfer.go:pluginCallError`).
- Egress policy handed to workers drops metadata and rule effects the worker does not apply (`egresspolicy.go:toAteletEgressPolicy`).
- A Checkpoint that detaches from the caller's context is bounded by `defaultCheckpointTimeout` (5 minutes) so a hung worker or upload cannot hold the actor lock forever (`main.go:defaultCheckpointTimeout`).
- Actor identity is barred from metric labels; per-actor snapshot phase records go to logs keyed by template only (`snapshotlog.go:snapshotLogAttrs`).
- /readyz flips to 503 on SIGTERM while /healthz stays 200; readiness also gates on the gRPC port being open (`main.go:servingCheck`, `main.go:readiness`).

## Side Effects

- Serves gRPC on TCP `--port` (to ate-api-server, mTLS from the podidentity credential bundle) and on the unix socket `nodepath.WorkerSupportSocket` (to workers on the node).
- Reads/writes the node filesystem: image cache, sandbox asset cache, snapshot cache and scratch, per-actor dirs and snapshot state, sandbox runtime index.
- Talks to Kubernetes APIs (informer factories for WorkerPools, SandboxConfigs, worker pods, ClusterTrustBundles, events) and ate-api-server (worker/control clients).
- Moves snapshot files through the object-store plugin socket; reads gs:// assets anonymously when the plugin cannot serve them; pulls images via the image cache with kubelet credential-provider plugins.
- Runs the OTLP relay on a unix socket for worker telemetry; registers device plugins with kubelet; starts GC and background worker goroutines; emits OTel metrics.

## Verification

- Run: `go test ./cmd/atelet/...`
