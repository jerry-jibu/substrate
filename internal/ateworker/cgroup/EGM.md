# cgroup

## Overview

Linux-only package that delegates the worker pod's cgroup v2 scope so each actor gets a leaf cgroup of its own, with CPU quota, `memory.high` caps, and a kill path. It also watches a leaf's memory pressure to decide when a throttled actor is stuck and should be killed rather than left stalling (runc/systemd `OOMPolicy=kill` would otherwise take down the whole worker).

## Interface

- `cgroup.go:Delegate(ctx)` - prepares the pod's cgroup scope: moves worker processes into a `worker` leaf, enables delegated controllers; `cgroup.go:Root` is `/sys/fs/cgroup`.
- `actor.go:OpenActorLeaf(actorUID, milliCPU, memoryHigh) (*ActorLeaf, error)` - creates or reuses the actor's leaf and caps it.
- `actor.go:ActorLeaf.SysProcAttr()` - starts a process directly inside the leaf via `UseCgroupFD`.
- `actor.go:LimitActorLeafGrowth(actorUID, headroom, ceiling, borrow)` / `actor.go:LiftActorLeafMemory(actorUID)` - set or clear `memory.high`, returning a restore func.
- `actor.go:ActorLeafMemory(actorUID)` - reads leaf `memory.current`; `actor.go:PodMemory()` - reads pod `memory.max` and unreclaimable usage.
- `actor.go:KillActorLeaf(ctx, name)` / `actor.go:KillLeafUnder(ctx, root, name)` - write `cgroup.kill` and wait for `populated 0`.
- `actor.go:RemoveActorLeaf(actorUID)` - deletes the empty leaf.
- `memguard.go:MemoryGuard` with `Check(ctx, root, name) (Kill, bool)`, `Killed()`, `Pause()`, `Resume(limit)`; `memguard.go:MemoryGuardInterval`; `memguard.go:PSIEnabled(root)`.
- `memwatch.go:MemoryWatch.Check(dir, limit, now)` - reports whether the cgroup is stuck at its limit.

## Invariants

- `Delegate` fails unless `/proc/self/cgroup` reads `0::/`, i.e. the worker runs unprivileged in a private cgroup namespace (`cgroup.go:delegate`).
- Controllers are enabled one at a time so one undelegatable controller does not block the rest (`cgroup.go:delegate`).
- `actor.go:actorLeafPath` rejects empty, `.`, `..`, the reserved `worker` leaf name, and names containing `/` or NUL.
- `cpu.max` and `memory.high` are rewritten even when unset, so a reused leaf keeps no old cap; a missing file (undelegated controller) is tolerated (`actor.go:openActorLeaf`).
- A MemoryGuard does nothing until armed via `Resume`; a paused, killing, or already-killed guard does nothing (`memguard.go:MemoryGuard.Check`).
- A stuck cgroup is judged from memory stalls (`memory.pressure` some total over a 2s window, >= 25%), its own `memory.events.local high` count, and usage within 1/16 of the limit; page cache (dirty snapshot pages) is excluded via memory.stat (`memwatch.go:MemoryWatch.Check`).
- `writeExisting` never creates cgroup interface files, which cgroupfs refuses with EACCES (`actor.go:writeExisting`).

## Side Effects

- Bind-remounts `/sys/fs/cgroup` read-write when the runtime mounted it read-only (`unix.Mount` in `cgroup.go:delegate`).
- Moves worker PIDs into the `worker` leaf by writing `cgroup.procs` (`cgroup.go:moveProcs`).
- Writes cgroup v2 files: `cgroup.subtree_control`, `cpu.max`, `memory.high`, `cgroup.kill` (kills every process in the leaf).
- Spawns a goroutine per guard kill with a 10s timeout (`memguard.go:MemoryGuard.Check`).
- Reads `memory.current`, `memory.stat`, `memory.pressure`, `memory.events.local`, `memory.max`, `cgroup.events` on a timer.

## Verification

- Run: `go test ./internal/ateworker/cgroup/...`
