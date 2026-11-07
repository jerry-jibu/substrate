# workloads

Deploys the benchmark WorkerPool and ActorTemplates that the locust and
nighthawk benchmarks run against. One script, `deploy.sh`, drives everything;
the definitions live under `manifests/`.

## Interface

- `deploy.sh` - `--deploy` / `--delete` the workload set. Sizing flags: `--worker-count` (default 1), `--sandbox-class` `gvisor|microvm`, `--actor-memory` (default 256Mi) and `--actor-cpu` (default 1, converted to millicores by `cpu_millicores`), `--worker-memory`, `--wait-timeout` (default 300 s).
- `deploy.sh` template selection - default set `sleep glutton glutton-durdir-data glutton-durdir-full`, overridable with `WORKLOAD_TEMPLATES`; `glutton-storage` joins only when `--storage-class-name` is set, `ate-env` only when `ATE_ENV_IMAGE` is set; `sweperf-*` names resolve from `manifests/sweperf/generated/` (`deploy.sh:template_manifest`).
- `deploy.sh:resolve_otlp_endpoint` - reads `OTEL_EXPORTER_OTLP_ENDPOINT` from the `ate-otel-config` ConfigMap in `ate-system`, so actor telemetry follows the cluster's collector; `--otlp-endpoint` overrides.
- `deploy.sh:wait_actortemplate_ready` / `deploy.sh:wait_templates_ready` - poll each template's `status.goldenSnapshotStatus` until a golden tag exists, failing fast on `errorMessage`.
- `deploy.sh:fake_data_plane` / `deploy.sh:tune_fake_atelet` - on an install with `--fake-data-plane`, write `FAKE_ATELET_<FLAG>=VALUE` entries to the `fake-atelet-tuning` ConfigMap and restart the atelet DaemonSet when it changes.
- Subdirectory contract: [manifests](./manifests/EGM.md).

## Invariants

- Every requested template must have a manifest before anything on the cluster is deleted or created (`deploy.sh:deploy` pre-check).
- ActorTemplates are immutable (no update RPC), so deploy deletes and recreates each template; that removal also drops the old golden actor and snapshot server-side (`deploy.sh:deploy` comment).
- The sandbox class is validated to `gvisor` or `microvm` before any action runs (`deploy.sh` argument tail).
- `--fake-atelet-env` entries must match `FAKE_ATELET_[A-Z_]+=.+` and require an install with `--fake-data-plane` (`deploy.sh` argument tail, `deploy.sh:deploy`).
- The store enforces that a template's atespace exists at create time, so `benchmark-workloads` is created before the templates (`deploy.sh:deploy`).
- An unreadable `ate-otel-config` is an error, not a default: a fallback endpoint would send actor telemetry to the wrong collector silently (`deploy.sh:resolve_otlp_endpoint` comment).

## Side Effects

- Builds `kubectl-ate` into a temp dir, runs `ko apply`/`ko resolve`/`ko delete`, `kubectl-ate` atespace/actor-template create and delete, `kubectl wait`/`rollout`, and on the fake data plane rewrites the `fake-atelet-tuning` ConfigMap and restarts the atelet DaemonSet (`deploy.sh:deploy`, `deploy.sh:delete`, `deploy.sh:tune_fake_atelet`).

## Verification

- Run: `hack/verify/shellcheck.sh` (shellchecks every repo `.sh` script).
