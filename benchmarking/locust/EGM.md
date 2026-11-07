# locust

The locust-based load generator for substrate benchmarks. `runner.py` runs a
headless locust test, spawns the Go `boomer-worker` for boomer user classes,
and publishes stats as JSONL + CSVs plus server-side telemetry to
`<dest>/runs/<name>/run_date=.../run_ts=.../run_tag=.../`. The Dockerfile
bakes locust, the generated Python gRPC clients, and the boomer binary into
one distroless image; `manifests/locust.yaml` (rendered by `deploy.sh`)
launches it in the `benchmarking` namespace.

## Interface

- `runner.py:parse_args` / `runner.py:main` - the run entry point: `-f` (comma-separated test file then shape), `-t`, `-u`, `--tag`, `--name`, `--dest` (gs:// or local), `--allow-empty-stats`, `--actors-per-user`, spawn flags, `--cluster-facts`, `--metadata`, `--prometheus-url`, `--atelet-lag-s`, `--phase-breakdown`.
- `runner.py:PYTHON_TESTS` / `runner.py:needs_boomer` / `runner.py:test_file` - the closed set of Python-implemented tests; every other test runs on boomer, and `test_file` picks the non-shapes path from `-f`.
- `runner.py:run_test` - runs locust `--headless` with `--csv-full-history`, in `--master --expect-workers 1` mode with `--boomer-config-port 5560` when boomer is needed, spawns `/app/boomer-worker`, and drains it with SIGTERM (90 s) after locust exits.
- `runner.py:stats_to_jsonl` / `runner.py:upload` / `runner.py:upload_to_gcs` - stats CSV to per-operation JSONL rows, and upload to GCS or local disk.
- `cluster_facts.py:get_cluster_hardware_facts` / `cluster_facts.py:append_trial_summary` / `cluster_facts.py:snapshot_worker_pods` / `cluster_facts.py:read_evicted_pods` / `cluster_facts.py:diff_worker_pods` / `cluster_facts.py:EMPTY_FACTS` - node capacity, worker pod count, density frontiers, and OOM/eviction facts from the Kubernetes API.
- `server_telemetry.py:extract_and_record_server_telemetry` - Prometheus range queries over the run window: cluster packing, node/pod PSI, snapshot sizes and latencies, active actors, OOM kills (`server_telemetry.py` module docstring).
- `phase_breakdown.py:append_phase_breakdown` / `phase_breakdown.py:read_pod_logs` - suspend/resume phase percentiles from atelet and worker pod logs, appended to stats.jsonl.
- `ladder_steps.py:append_ladder_steps` - one `ladder_step_NNN` stats row per ladder step, querying Prometheus over each step's steady window (`SETTLE` 30 s / `TAIL` 5 s).
- `turnover_window.py:append_turnover_window` / `turnover_window.py:fill_done_ts` - the `turnover_window` row for an agentsession turnover run, measured from the `"agentsession: turnover fill done"` log line.
- `deploy.sh` - `--deploy` / `--delete` the locust master and workers via `envsubst` on `manifests/locust.yaml`; `--user-class` picks `tests/<name>.py`; `--agentsession-script FILE` validates, uploads, and mounts a script ConfigMap.
- `build_and_push.sh` - builds and pushes the locust image (`LOCUST_IMAGE`, or `${KO_DOCKER_REPO}/locust-test:latest`); platform must match the cluster nodes, not the build host.
- Subdirectory contracts: [codegen](./codegen/EGM.md), [common](./common/EGM.md), [shapes](./shapes/EGM.md), [unit_tests](./unit_tests/EGM.md). Also `tests/` (the locust user classes, no contract) and `manifests/locust.yaml`.

## Invariants

- The post-run steps (cluster facts, server telemetry, ladder steps, turnover window, phase breakdown) are additive: each is wrapped so a failure logs a warning and never discards the locust measurements (`runner.py:main`).
- A run with no stats rows exits 1 unless `--allow-empty-stats` was given (`runner.py:main` tail).
- New load generators are written in boomer, not Python; `PYTHON_TESTS` is a closed set to shrink as its entries retire (`runner.py:PYTHON_TESTS` comment).
- The phase breakdown and turnover reads happen before this process exits, because the orchestrator deletes the worker and ate-system pods as soon as it does (`phase_breakdown.py` module docstring, `runner.py:main`).
- Evictions are read right after the run because Evicted events expire after about an hour (`runner.py:main` comment).
- The Dockerfile's build context must be the monorepo root; the image carries both locust and `/app/boomer-worker`, built `CGO_ENABLED=0` for boomer's pure-Go ZMQ (`Dockerfile`).

## Side Effects

- Writes a `/tmp/<timestamp>-locust-runner` work dir and uploads status.json, logs.txt, traces.txt, stats.jsonl, CSVs, and server_summary.json to GCS (`google.cloud.storage`) or local disk (`runner.py:main`, `runner.py:upload_to_gcs`).
- Reads the Kubernetes API: nodes, worker pods, atelet/worker pod logs, and Evicted events (`cluster_facts.py`, `phase_breakdown.py`).
- Queries Prometheus over HTTP (`server_telemetry.py`, `ladder_steps.py`).
- SIGTERM to boomer, which suspends and deletes every actor it created (`runner.py:run_test`).
- `deploy.sh` creates/labels the `benchmarking` namespace (`ate.dev/api-access=true`), applies the rendered manifest, and manages the `agentsession-script` ConfigMap (`deploy.sh:deploy`, `deploy.sh:delete`).

## Verification

- Run: `hack/verify/benchmarking-python.sh` (runs the `unit_tests` suite from this directory).
