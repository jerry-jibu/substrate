# fake-workersync

## Overview

Stands in for atecontroller's WorkerPool controller and worker syncer in
control-plane benchmarks. Honors every WorkerPool's `spec.replicas` with
fake Workers no pod backs, placed round robin on nodes running a
fake-atelet pod, and reports each Worker's capacity through the fake-atelet
on its node, as worker reports through atelet. Writes each pool's status so
the scale subresource and `kubectl get workerpool` work.

## Interface

- `main.go:main` - flags: `--api-endpoint`, `--run` (default `bench`), `--atelet-namespace`, `--fake-atelet-label`, `--relay-port` (default `fakeworker.DefaultRelayPort` 8086), `--atelet-spiffe-id`, `--client-cred-bundle`, `--trust-bundle`, `--resync`, `--concurrency` (32), `--cleanup`, `--node-grace`, `--health-listen-addr` (`:9091`), `--drain-grace`, `--version`, `--log-level`, `--preview`.
- `main.go:kubeCluster` - controller's view of Kubernetes: `Pools`, `Nodes` (within `--node-grace` of a fake-atelet pod), `Relays` (running fake-atelet pod IPs by node), `UpdateStatus`.
- `main.go:healthHandler` - `/healthz` always 200; `/readyz` 200 only after adopt succeeds.
- `controller.go:controller` - keeps registered Workers in memory; `controller.go:adopt` lists Workers of this run at startup; `controller.go:reconcile` creates/retires Workers and syncs status; `controller.go:deleteAll` for `--cleanup`.
- `controller.go:workerClient` - the subset of `ateapi.ControlClient` used: `CreateWorker`, `DeleteWorker`, `DrainWorker`, `MarkWorkerAvailable`, `GetWorker`, `UpdateWorker`, `ListWorkers`, `ListWorkerActorAssignments`.
- `controller.go:templateOf`/`template.capacity`/`fits`/`maxActors` - per-pool worker template (sandbox class, cpu/memory limits, actor count from `fakeworker.MaxActorsAnnotation`).
- `relayclient.go:grpcRelay` - sends capacity reports to the fake-atelet relay via `WorkerService` RPCs `AcquireWorkerEpoch`, `RegisterWorker`, `SetWorkerSandboxRuntimes`; `relayclient.go:grpcRelay.Prune` drops connections to gone relays.
- Subdirectories: [../internal/EGM.md](../internal/EGM.md) - shared fake-worker naming and identity checks.

## Invariants

- Workers left registered across a restart are adopted, not recreated; a restarted fake-workersync reuses the Worker's registered epoch so ate-api-server does not crash Actors on it (`main.go:main` doc comment, `controller.go:workerReport.Epoch`).
- atecontroller must not run alongside: its controller would create real pods for the same pools and its syncer deletes Workers with no live pod (`main.go:main` doc comment).
- Every reconcile step is idempotent; a steady fleet makes no ate-api-server calls (`controller.go:reconcile` doc comment).
- A Worker is tracked before `CreateWorker` and marked created after, so a create the server committed but the caller saw fail is still retired or cleaned up; retire/deleteAll treat NotFound as done (`controller.go:ensure`).
- A pool whose spec cannot be read is held: its Workers stay rather than being retired (`controller.go:reconcile` `held`).
- Capacity is fixed when the Worker is made and reported once; a Worker registered with other IPs or capacity takes a new epoch (`controller.go:ensure`).
- Retire drains first, then deletes after `--drain-grace` even with Actors still assigned; `DeleteWorker` releases them (`controller.go:retire`).
- The relay connection verifies the server chain against the pod-identity trust bundle and its SPIFFE ID (`relayclient.go:newGRPCRelay` via `fakeworker.VerifyPeerID`); the trust bundle is re-read at each handshake.
- `--cleanup` serves no health endpoints and requires the serving instance to be stopped first (`main.go:--cleanup` flag doc, `main.go:main`).

## Side Effects

- Calls ate-api-server Control RPCs through `boomerutil.DialControl` (`main.go:main`).
- Watches WorkerPools with a shared informer and writes WorkerPool status subresources (`main.go:main`, `controller.go:syncStatus`).
- Opens mTLS gRPC connections (TLS 1.3, client cert from `--client-cred-bundle`) to fake-atelet relays by pod IP (`relayclient.go:newGRPCRelay`, `relayclient.go:grpcRelay.conn`).
- Serves `/healthz` and `/readyz` on `--health-listen-addr` (`main.go:healthHandler`).

## Verification

- Run: `go test ./cmd/benchmarking/isolate/fake-workersync/...`
