# actorlock

## Overview
Per-actor serialization for lifecycle RPCs. `Locks` splits locking by actor so one actor's activation never waits on an unrelated actor's, and `InFlight` tracks running RPCs for shutdown cancellation and draining.

## Interface
- `actorlock.go:CancelableMutex`, `actorlock.go:NewCancelableMutex` - mutex with `Lock`, `Unlock`, and `LockContext` that returns false on cancellation without acquiring.
- `actorlock.go:Locks`, `actorlock.go:New` - per-actor lock map; `Lock(ctx, actorUID)`, `Busy(actorUID)`, `Unlock(actorUID)`. Zero value is ready to use.
- `inflight.go:InFlight`, `inflight.go:NewInFlight` - registry of running RPCs; `Add` records an RPC and returns its clear function, `CancelStartups` cancels startup RPCs and returns their actor UIDs, `WaitIdle` waits for all RPCs to finish, `Names` returns the RPC name per active actor.

## Invariants
- Locks serialize RPCs for the same actor and never block unrelated actors (`actorlock.go:Locks`).
- `Busy` counts waiters as well as holders, and never blocks; its answer may be stale, for advisory checks only (`actorlock.go:Busy`).
- Unused lock entries are removed on release, so the map does not grow per actor ever hosted (`actorlock.go:release`).
- An already-canceled context never takes a free lock (`actorlock.go:LockContext`).
- Callers must serialize RPCs for each actor with `Locks` before using `InFlight` (`inflight.go:InFlight`).
- `Add` takes a nil cancel for an RPC that must run to completion, such as a checkpoint saving state (`inflight.go:Add`).

## Side Effects
- None beyond in-memory synchronization; no disk, network, or process effects.

## Verification
- Run: `go test ./internal/actorlock/...`
