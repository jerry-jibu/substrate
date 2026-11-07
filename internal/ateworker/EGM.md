# ateworker

## Overview

Shared library code for the worker runtimes (`cmd/ateworker-gvisor`, `cmd/ateworker-microvm`) that host actor sandboxes on a node. The package root holds checkpoint completion/reuse logic; the child packages cover actor networking, cgroup delegation, memory borrowing, phase logging, PID 1 supervision, worker registration with atelet, and upward suspend requests.

## Interface

- `checkpoint.go:CheckpointContext(ctx) (context.Context, context.CancelFunc)` - keeps ctx's values and deadline but not its cancellation, so atelet going away does not cancel a checkpoint that may have stopped the workload.
- `checkpoint.go:RecordCheckpoint(checkpointDir, bootID string, fidelity ateworkerpb.SnapshotFidelity, resp *ateworkerpb.CheckpointWorkloadResponse)` - writes the `.checkpoint-done.json` completion marker (via temp file and rename).
- `checkpoint.go:ReuseCheckpoint(checkpointDir, bootID string, fidelity ateworkerpb.SnapshotFidelity) *ateworkerpb.CheckpointWorkloadResponse` - returns the recorded response when the marker matches boot ID and fidelity and all listed files are still present.
- Subdirectory contracts:
  - [actornet](./actornet/EGM.md) - per-actor private `/30` network namespaces, nftables egress redirect, and sandbox `/etc` files.
  - [cgroup](./cgroup/EGM.md) - cgroup v2 delegation, per-actor leaves, memory guard and kill.
  - [memborrow](./memborrow/EGM.md) - ledger lending unbooked pod memory to snapshot writes.
  - [phaselog](./phaselog/EGM.md) - shared shape of per-actor checkpoint/restore phase log records.
  - [pid1](./pid1/EGM.md) - minimal init that reaps orphans for worker containers.
  - [register](./register/EGM.md) - worker registration and sandbox runtime reporting via atelet, plus file watching and crash reports.
  - [suspend](./suspend/EGM.md) - upward actor suspend requests via atelet.

## Invariants

- A checkpoint runs detached from its caller's cancellation but keeps its deadline (`checkpoint.go:CheckpointContext`).
- The completion marker records the node boot ID; the worker does not sync snapshot files, so after a reboot they may be torn and the marker must not be reused (`checkpoint.go:checkpointRecord`).
- Reuse requires matching boot ID, matching fidelity, `DataSnapshotFiles` being a subset of `SnapshotFiles` (equal on VOLUMES, strict on ROOTFS/MEMORY), and every listed file present and regular (`checkpoint.go:ReuseCheckpoint`).

## Side Effects

- Writes and renames `.checkpoint-done.json` inside the checkpoint directory; the directory is wiped by `os.RemoveAll` on the next checkpoint or atelet's `resetActorDirs` (`checkpoint.go:RecordCheckpoint`).

## Verification

- Run: `go test ./internal/ateworker/...`
