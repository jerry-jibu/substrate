# fakeworker

## Overview

What the two programs of the isolate benchmark's fake data plane share: how
a fake Worker's stand-in pod is named and placed, and how each side checks
the other's identity on the capacity relay between them.

## Interface

- `fakeworker.go:ValidateRun` - run prefix must be 1 to `fakeworker.go:MaxRunLength` (8) lowercase letters or digits.
- `fakeworker.go:Prefix`, `fakeworker.go:Name`, `fakeworker.go:Index` - pod naming: `fake-<run>-<sha256(namespace/pool)[:8]>-<index>`; `Index` reverses it.
- `fakeworker.go:Node` - round-robin placement of fake worker index over sorted nodes.
- `fakeworker.go:IP` - documentation-range address `192.0.2.<n>` per index, repeating past 254 workers.
- `fakeworker.go:VerifyPeerID` - checks the peer's leaf certificate URI (SPIFFE ID) equals the expected value, complementing chain verification.
- Constants: `fakeworker.go:DefaultMaxActors` (1000), `fakeworker.go:MaxActorsAnnotation` (`ate.dev/fake-max-actors`), `fakeworker.go:WorkerPoolLabel` (`ate.dev/worker-pool`), `fakeworker.go:DefaultRelayPort` (8086).
- Subdirectories: [./fakeworkertest/EGM.md](./fakeworkertest/EGM.md) - test-only CA and SPIFFE-shaped certificate issuer.

## Invariants

- Generated pod names stay inside the 63-character Kubernetes limit; the pool name enters as an 8-hex-char hash because namespace and name together can exceed it (`fakeworker.go:Name` doc comment, `fakeworker.go:poolPrefix`).
- `Index` accepts only canonical non-negative decimal suffixes (`fakeworker.go:Index` rejects `strconv.Atoi` mismatches).
- `VerifyPeerID` proves which workload, not the signer; chain verification against the pod-identity trust bundle is separate (`fakeworker.go:VerifyPeerID` doc comment).

## Side Effects

- ["None"]

## Verification

- Run: `go test ./cmd/benchmarking/isolate/internal/fakeworker/...`
