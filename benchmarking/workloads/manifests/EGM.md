# manifests

The benchmark workload definitions: `workloads.yaml.tmpl` (the
`benchmark-workloads` namespace plus the `benchmark-worker` WorkerPool) and
one protojson-shaped `ateapi.ActorTemplate` per workload, substituted by
`../deploy.sh` and created through the ate API with `kubectl-ate create
actor-template`. The templates are not CRDs.

## Interface

- `workloads.yaml.tmpl` - Namespace `benchmark-workloads` and WorkerPool `benchmark-worker`: `${WORKER_COUNT}` replicas, `${SANDBOX_CLASS}` (gvisor or microvm), `ko://go.ate.dev/substrate/cmd/ateworker-${SANDBOX_CLASS}` image, a toleration letting gvisor and microvm workers share `ate.dev/sandboxClass=shared` nodes, and `${WORKER_RESOURCES}`.
- `glutton-template.yaml.tmpl` - the glutton workload: `cmd/benchmarking/glutton` on `:80` (HTTP mode) and metrics on `:9090`, `--data-dir=/var/lib/glutton`, OTLP endpoint `${OTLP_ENDPOINT}` with `service.name=glutton-actor`, wakeup probe on `/readyz:80`, `${ACTOR_CPU_MILLICORES}`/`${ACTOR_MEMORY_BYTES}` limits, `SNAPSHOT_FIDELITY_MEMORY`.
- `glutton-durdir-data-template.yaml.tmpl` / `glutton-durdir-full-template.yaml.tmpl` - the DurDir variants: suspend uploads only the durable dir (DATA-scope commit; resume is a cold boot) vs FULL-scope commits.
- `glutton-storage-template.yaml.tmpl` - glutton with an external volume backed by `${STORAGE_CLASS_NAME}`; run with `--durdir-template glutton-storage`.
- `sleep-template.yaml.tmpl` - busybox `/bin/sleep 1000000` for a minimal snapshot payload.
- `usermem-template.yaml.tmpl` - `stress` holding 1 GiB of user memory (for `locust/tests/usermem.py`), fixed 1536Mi limit.
- `kernelmem-template.yaml.tmpl` - dentry-cache fill plus 50000 sleeping processes (~6.3 GiB kernel state, for `locust/tests/kernelmem.py`), fixed limit.
- `ate-env-template.yaml.tmpl` - runs `${ATE_ENV_IMAGE}` to dump the actor environment.
- Subdirectory contract: [sweperf](./sweperf/EGM.md).

## Invariants

- Every template selects workers with `workload: benchmark-worker` and lives in the `benchmark-workloads` atespace, which deploy.sh creates before the templates because the store enforces its existence at create time (`../deploy.sh:deploy`).
- `usermem` and `kernelmem` are not in deploy.sh's default template set and pin their memory limits instead of taking `${ACTOR_MEMORY_BYTES}` (`usermem-template.yaml.tmpl` header, `kernelmem-template.yaml.tmpl` header).
- Glutton's port 80 is the only inbound port nftables forwards from the worker pod into the actor sandbox, and it must serve HTTP/1.1 for the router (`glutton-template.yaml.tmpl` command comment).
- Keep the sleep template's memory small: whatever the micro-VM guest kernel caches rides along in the memory snapshot (`sleep-template.yaml.tmpl` resources comment).
- Without `OTEL_EXPORTER_OTLP_ENDPOINT` in the template, the actor's OTel SDK sends spans to localhost:4317 where nothing listens, and substrate discards them silently (`glutton-template.yaml.tmpl` env comment).

## Side Effects

- Consumed by `../deploy.sh`, which substitutes `${...}` values, resolves `ko://` image references, and applies them to a cluster (`../deploy.sh:substitute`, `../deploy.sh:deploy`).

## Verification

- Run: `hack/verify/benchmarking-python.sh` (covers `sweperf/`); the `.tmpl` files themselves are checked only by deployment.
