# podcertcontroller/internal/rendezvous

## Overview
Rendezvous hashing on top of Kubernetes Leases so multiple controller replicas agree on which replica handles an item. Unlike leader election it spreads work across all live replicas and rebalances as replicas come and go.

## Interface
- `rendezvous.go:Hasher` - the lease-backed rendezvous hasher (FNV-1a 64-bit).
- `rendezvous.go:New` - builds a Hasher from a kubernetes client, namespace, application name, replica name/UID, and clock.
- `rendezvous.go:LeaseDuration` - 15s; upper bound on how long a dead replica keeps its assignments.
- `rendezvous.go:ErrNotAssigned` - sentinel error for items not assigned to this replica.
- `Hasher.Run` / `Hasher.AssignedToThisReplica` - heartbeat loop and assignment query (consumed by `signercontroller.Hasher` and `cmd/podcertcontroller/main.go`).

## Invariants
- Assignment is not atomic: controllers' actions must be safe even if multiple replicas process the same item simultaneously (`rendezvous.go` package doc).
- A replica is considered live only while its Lease has been renewed recently. This package never deletes Leases; a dead replica's Lease is reclaimed by Kubernetes garbage collection through its Pod `OwnerReference` (`rendezvous.go:163-171`).
- Replicas may briefly disagree about live membership during changes; affected items can be ignored for up to one lease timeout (`rendezvous.go:Hasher` doc).

## Side Effects
- Writes and renews Lease objects (every 10s renewal period) in the configured namespace; it does not delete Leases.
- Runs a Lease informer watching leases labeled `rendezvous.ate.dev/application`.

## Verification
- Run: `go test ./cmd/podcertcontroller/internal/rendezvous/...`
