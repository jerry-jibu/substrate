# Agent Substrate Blueprint

Agent Substrate is a system built on top of Kubernetes that manages
agent-like workloads ("actors") to achieve higher scale and efficiency than
Kubernetes alone, with lower latency. It maps a larger set of actors onto a
smaller set of ready "workers" (Kubernetes Pods), taking the Kubernetes
control plane out of the critical path. It relies on agent-like applications
being idle most of the time to multiplex heavily.

The contract files in this tree follow the Elephant-and-Goldfish Model: leaf
directories document their real code surface (Interface), the state and call
rules that must hold (Invariants), and their externally visible writes
(Side Effects); parents roll their children up. Start at a top-level
directory below and descend.

## Subdirectories & System Boundaries

- [./api/EGM.md](./api/EGM.md) - public APIs: gRPC protobuf APIs (`ateapi`, `credprovider`, `objectstore`) and the `ate.dev/v1` Kubernetes CRD types.
- [./benchmarking/EGM.md](./benchmarking/EGM.md) - load-testing tools and workloads (locust, nighthawk, atunnel bench, suspend/resume analysis).
- [./cmd/EGM.md](./cmd/EGM.md) - one directory per shipped binary (`ateapi`, `atelet`, `atenet`, `atecontroller`, worker runtimes, `kubectl-ate`, ...).
- [./demos/EGM.md](./demos/EGM.md) - self-contained example applications, each with a workload binary, install templates, and tests.
- [./hack/EGM.md](./hack/EGM.md) - dev/CI shell scripts: verification, codegen, cluster lifecycle, e2e entry points.
- [./internal/EGM.md](./internal/EGM.md) - shared module-private packages: node and sandbox plumbing, atelet/worker protocols, e2e harness.
- [./manifests/EGM.md](./manifests/EGM.md) - Kubernetes YAML for deploying into `ate-system`.
- [./pkg/EGM.md](./pkg/EGM.md) - shared packages intended for external import (object-store plugin libraries, generated CRD clients).
- [./site/EGM.md](./site/EGM.md) - the Hugo documentation website and its build adapters.
- [./tools/EGM.md](./tools/EGM.md) - standalone Go dev/CI tools (`apitool`, `junittool`, ...).
- `docs/` - design docs and developer guides; documentation only, no contract.
- `vendor/`, `logo/`, `_LICENSES/` - vendored code and assets, outside the contract tree.

## Global Invariants

- Actors run strictly inside a sandbox: gVisor (`cmd/ateworker-gvisor`) or Cloud Hypervisor microVM (`cmd/ateworker-microvm`); plain Linux containers are not supported (AGENTS.md, `docs/security/security.md`).
- `atelet` and the worker runtimes are opaque to each other. The only shared knowledge is `internal/proto/ateworkerpb` and the paths in `internal/nodepath` (AGENTS.md "Code" section).
- Every served unary RPC of `ate-api-server` needs an entry in `defaultRPCPermissions` (`cmd/ateapi/internal/authz/registry.go`); an RPC without one is denied and a test fails the build.
- Outbound actor traffic traverses `atunnel` to `atenet-egress`, which enforces per-actor `EgressPolicy`; actors have no ambient credentials.
- From v1.0.0, changes to RPCs, CRDs, the worker pod template, database migrations, or node-shared files must keep upgrades and rollbacks working under `docs/reference/version-skew-policy.md`.
- Metrics and log events must be defined in the OpenTelemetry Weaver registry under `docs/metrics/registry/`; `hack/verify/metrics.sh` enforces it.

## Global Side Effects

- Installs and tests mutate Kubernetes clusters (`ate-system` namespace, per-`WorkerPool` NetworkPolicies, CRDs) and require one for e2e (`hack/run-e2e.sh`).
- The control plane persists state in PostgreSQL (`cmd/ateapi/internal/store/atepg`, schema set up by `pkg/postgressetup`) and stores snapshots in object storage (`pkg/objectstorage`, GCS/S3).
- Image builds and releases push to container registries via ko (`cmd/ate-setup`, `hack/release/`).

## System Verification

- Run: `go test -race ./...` from the repo root.
- Full linters and verifiers: `./hack/verify-all.sh`.
- E2E against a local Kind cluster: `./hack/run-e2e-kind.sh`.
- Root-gated packages (network namespaces, cgroups): `sudo ./hack/run-root-tests.sh`.
