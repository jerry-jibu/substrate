# benchmarking

Load-testing tools and workloads for Agent Substrate: locust-based actor
traffic, nighthawk ingress/egress benchmarks, the atunnel data-path
benchmark harness, suspend/resume timing analysis, and the reusable actor
workload images. `deploy_locust.sh` is the one-shot deploy/uninstall entry
point for the whole locust stack.

## Interface

- `deploy_locust.sh` - end-to-end deploy/uninstall of the locust benchmarking stack; flags `--deploy` and `--delete`, options `--worker-count`, `--sandbox-class` (default `gvisor`), `--skip-build`. Deploy order: `workloads/deploy.sh` → `locust/build_and_push.sh` → `locust/deploy.sh`; delete runs in reverse.
- Subdirectories (each has a contract): [./analysis/EGM.md](./analysis/EGM.md), [./atunnel/EGM.md](./atunnel/EGM.md), [./automation/EGM.md](./automation/EGM.md), [./locust/EGM.md](./locust/EGM.md), [./nighthawk-egress/EGM.md](./nighthawk-egress/EGM.md), [./nighthawk-ingress/EGM.md](./nighthawk-ingress/EGM.md), [./workloads/EGM.md](./workloads/EGM.md).

## Invariants

- `deploy_locust.sh` deploys workloads before the locust driver and deletes in the reverse order, so the driver never starts against missing workloads (`deploy_locust.sh` header comment).
- `automation/scratch/` is gitignored scratch space for benchmark automation; automation code must recreate it, not expect it to exist (`.gitignore`).

## Side Effects

- `deploy_locust.sh` builds and pushes container images and applies Kubernetes resources to the connected cluster (`deploy_locust.sh:main`).
- Benchmark runs write pod logs, `stats.jsonl`, and timing reports into cluster namespaces (`benchmark-workloads`) and local scratch directories (`./analysis/EGM.md`, `./automation/EGM.md`).

## Verification

- Run: `go test ./benchmarking/...` (Python dirs under `locust/` have their own unit tests; see [./locust/EGM.md](./locust/EGM.md)).
