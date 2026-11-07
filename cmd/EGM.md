# cmd

## Overview

One subdirectory per shipped binary of Agent Substrate. Each binary is a
thin `main.go` over packages in `internal/` (or its own `internal/`), so
the contracts here point at the binaries' entry points and their child
contracts where they exist.

## Subdirectories & System Boundaries

- ate-api-server: [./ateapi/EGM.md](./ateapi/EGM.md) - the Control and WorkerService gRPC servers.
- atecontroller: [./atecontroller/EGM.md](./atecontroller/EGM.md) - reconciles WorkerPools into worker pods and NetworkPolicies.
- ate-idp-server: [./ate-idp-server/EGM.md](./ate-idp-server/EGM.md) - serves actor JWT key discovery (JWKS).
- atelet: [./atelet/EGM.md](./atelet/EGM.md) - the node agent pairing workers with the control plane.
- ate-setup: [./ate-setup/EGM.md](./ate-setup/EGM.md) - cluster installer and image publisher (`go run ./cmd/ate-setup`).
- atenet: [./atenet/EGM.md](./atenet/EGM.md) - combined networking daemon (router, sdsmint).
- ateworker-gvisor: [./ateworker-gvisor/EGM.md](./ateworker-gvisor/EGM.md) - the gVisor sandbox worker runtime.
- ateworker-microvm: [./ateworker-microvm/EGM.md](./ateworker-microvm/EGM.md) - the Cloud Hypervisor microVM worker runtime.
- benchmarking: [./benchmarking/EGM.md](./benchmarking/EGM.md) - benchmark binaries (boomer-worker, benchmark-egress, glutton, isolate).
- credential-provider: [./credential-provider/EGM.md](./credential-provider/EGM.md) - credential provider plugins (kubernetes-secrets).
- dataplane: [./dataplane/EGM.md](./dataplane/EGM.md) - Envoy dataplane image sources for the atenet gateways.
- kubectl-ate: [./kubectl-ate/EGM.md](./kubectl-ate/EGM.md) - the `kubectl ate` plugin.
- objectstore-plugin: [./objectstore-plugin/EGM.md](./objectstore-plugin/EGM.md) - object-store storage plugin binary.
- podcertcontroller: [./podcertcontroller/EGM.md](./podcertcontroller/EGM.md) - issues pod identity certificates (podid, servicedns, postgres signers).

## Global Invariants

- Keep atelet and the worker runtimes opaque to each other; the only shared surface is `internal/proto/ateworkerpb` and the paths in `internal/nodepath` (AGENTS.md project rule, mirrored by the worker RPCs in `internal/proto/ateworkerpb`).
- Binaries stay thin: each `cmd/<name>/main.go` only wires flags and registers services implemented under `internal/` (see `cmd/atenet/main.go:main`, `cmd/ateworker-microvm/main.go:main`).

## Global Side Effects

- Every binary listens on a network socket or unix socket, or drives cluster state through the Kubernetes and gRPC APIs; specifics are in each child contract.

## System Verification

- Run: `go test ./cmd/...`
