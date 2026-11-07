# podcertcontroller/internal/signercontroller

## Overview
Generic in-memory signing controller for PodCertificateRequests. Drives a work queue fed by the PCR informer and delegates each request to a `SignerImpl`, publishing desired ClusterTrustBundles in the background.

## Interface
- `signercontroller.go:SignerImpl` - interface each signer implements: `SignerName()`, `DesiredClusterTrustBundles()`, `MakeCert(ctx, pcr)`.
- `signercontroller.go:Hasher` - work-sharding interface: `AssignedToThisReplica(ctx, item) bool`.
- `signercontroller.go:Controller` - the queue-driven controller.
- `signercontroller.go:New` - builds the Controller and registers add/update/delete event handlers on the PCR informer.
- `signercontroller.go:Run` - waits for informer sync, then runs the given number of worker goroutines plus a periodic bundle-ensure loop (`signercontroller.go:ensureBundles`, every 5s).

## Invariants
- `signercontroller.go:Run` forces at least one worker even if given fewer.
- The caller must start the informer separately; `signercontroller.go:Run` only waits for its sync.

## Side Effects
- Mutates PCR status via `podcertificate.Client.UpdateStatus` and reconciles ClusterTrustBundles through `clustertrustbundle.Client`.
- Enqueues on every PCR add/update/delete event (`signercontroller.go:New`).

## Verification
- Run: `go test ./cmd/podcertcontroller/internal/signercontroller/...`
