# automation

The benchmark orchestrator: it owns the generic lifecycle (clone, deploy
substrate, deploy workloads, submit a runner Job per test, tear down) while
each test type supplies its hooks from `testtypes/`. Runs as a container
image, driven by a CronJob that `setup.sh` generates, with the test list and
target-cluster configs mounted from ConfigMaps.

## Interface

- `orchestrator.py:parse_args` / `orchestrator.py:main` - `--repo`, `--branch`, `--dest`, `--tests` (default `/etc/orchestrator/tests.yaml`), repeatable `--test-name`, `--target-cluster-dir`, `--manifests-dir`, `--junit-output`.
- `orchestrator.py` lifecycle - `validate_and_normalize_tests`, `deploy_substrate` / `teardown_substrate`, `deploy_workloads` / `teardown_workloads`, `run_test`, `write_junit_xml`; per test it sources the target cluster env (`source_env` / `apply_target_cluster` / `clear_target_cluster`), builds the runner image once per cluster, renders the Job template (`render_template`), submits it, waits (`wait_for_job`, `wait_for_no_active_runners`), tails logs, and deletes it.
- `orchestrator.py:substrate_args` - the ate-setup flags for a test, including the isolate block's `--fake-data-plane` via `util.isolate_install_args`.
- `util.py:run` / `util.py:run_no_check` - logged subprocess wrappers; `util.py:build_and_push` - `docker build --platform linux/amd64` + push from the repo root.
- `util.py:ISOLATE_TUNING` / `util.py:isolate_block` / `util.py:isolate_workload_args` - the `isolate:` tests.yaml block: keys map to `FAKE_ATELET_*` tuning, validated (durations as fake-atelet parses them, `mintActorCertificate` bool) before any cluster work.
- `controlplane.py:validate` / `controlplane.py:config` / `controlplane.py:shape` - the `controlPlane:` block: `apiServer:` knobs (`cpu` required, `memory`, `replicas`, `poolMaxConns`), `postgres:` over defaults (`stats` true, `args`), applied by patching `ate-system`.
- `controlplane.py:set_pool_max_conns` / `controlplane.py:resize_api_server` / `controlplane.py:enable_postgres_stats` - the concrete patches (env ConfigMap `ate-api-server-envvars`, Deployment resize, postgres_exporter sidecar with `pg_stat_statements`).
- `setup.sh` - interactive wizard: snapshots `.ate-dev-env.sh` per target cluster into `scratch/target-clusters/`, builds and pushes the orchestrator image, and generates `scratch/cronjob.yaml`, `scratch/test-list.yaml`, and `scratch/target-clusters.yaml`.
- `tests.yaml` - the mounted test list; `manifests/` holds the runner Job and CronJob templates (no contracts).
- Subdirectory contract: [testtypes](./testtypes/EGM.md).

## Invariants

- Every entry is validated (type hooks plus `validate_and_normalize_tests`) before any other work, so a bad entry fails up front (`orchestrator.py` module docstring).
- Each test tears substrate and workloads down afterward and drops the env file, so tests cannot pollute each other and the next test cannot inherit this one's cluster (`orchestrator.py` module docstring step 3d).
- `apply_target_cluster` returns to the snapshotted `_ORIG_ENV` baseline before sourcing the next config, so stale vars do not carry over (`orchestrator.py:_ORIG_ENV` comment).
- An `isolate:` block with an unknown key or a value fake-atelet would reject fails before any cluster work, not in a crash-looping fake-atelet (`util.py:isolate_block`).
- Control-plane isolation from the load is the install's job (`--cordon-system-infra-components` in ateArgs), not shape()'s (`controlplane.py` module docstring).
- The exit is non-zero if any test did not complete (`orchestrator.py` module docstring step 4).

## Side Effects

- Clones the substrate repo, runs gcloud/docker/kubectl, deploys and deletes ate-system and the benchmark workloads, creates and deletes Jobs in the `benchmarking` namespace, and patches Deployments/ConfigMaps in `ate-system` (`orchestrator.py`, `controlplane.py`).
- `util.py:build_and_push` pushes runner images to the cluster's registry; `setup.sh` writes the scratch manifests and pushes the orchestrator image.

## Verification

- Run: `hack/verify/benchmarking-python.sh` (runs this directory's pytest suite).
