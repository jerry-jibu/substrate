# nighthawk-ingress

Nighthawk adaptive router-capacity benchmark. One Kubernetes Job per
`tests.yaml` entry: it creates and warms a fleet of glutton actors, drives
POST `/ping` traffic through `atenet-router` with
`nighthawk_adaptive_load_client`, and uploads results in the same
Hive-partitioned layout as the locust runner. The Job exits 0 iff the session
converged (client exit 0 and a testing-stage result parsed) and all artifacts
uploaded.

## Interface

- `runner.py:parse_args` / `runner.py:main` - the Job entry point: `--name`, `--tag`, `--dest`, `--envoy-cpu`, `--actors` (default 100), client sizing (`--client-concurrency` 16, `--connections` 1000, `--max-pending` 10000), search knobs (`--initial-rps` 500, `--exp-factor` 2.0, stage durations), thresholds (`--success-rate-threshold` 0.999, `--send-rate-threshold` 0.9, `--tail-latency-slo-ms`), `--router-url`, `--warm-deadline`, `--atespace`.
- `spec.py:build_spec_dict` - assembles the `nighthawk.adaptive_load.AdaptiveLoadSessionSpec` dict: per-actor `ate-target-actor` request source, permissive failure predicates, mTLS `tls_context` against `atenet-router.ate-system.svc`, exponential-search step controller, and binary scoring thresholds.
- `spec.py:spec_dict_to_textproto` / `spec.py:load_pool` / `spec.py:message_class` - validate the dict against the real protos from the descriptor set and render textproto.
- `actors.py:ateapi_stub` / `actors.py:_token_channel` - ControlStub over the pod-certificate mTLS channel, or over a bearer token from `/run/ateapi-token/token` when `ATE_ATEAPI_CLIENT_AUTH=token` (`runner.py:main` reads the env var).
- `actors.py:create_and_warm` / `actors.py:_warm_actor` - create `nh-<idx>` actors from `benchmark-workloads/glutton` and warm each by POSTing `/ping` through the router until 200, 16 in parallel.
- `actors.py:cleanup` - best-effort `DeleteActor` for the created fleet; never raises.
- `output.py:parse_output_textproto` - the session output textproto to a JSON dict via the descriptor pool.
- `output.py:stage_rows` / `output.py:stats_records` - one row per adjusting/testing stage, shaped as locust-pipeline-compatible stats.jsonl records.
- `output.py:capacity_summary` - the one-record verdict: `slo_max_rps` (achieved-rps of the highest fully-clean stage), `binding_threshold`, `converged`, testing-stage percentiles.
- `runner.py:start_service` / `runner.py:run_adaptive_client` - run `nighthawk_service` on `127.0.0.1:8443` (30 s readiness wait) and `nighthawk_adaptive_load_client` against it.
- `Dockerfile` - the runner image: descriptor build (see protogen), ateapi Python clients, pinned `envoyproxy/nighthawk-dev` base; build with `--platform linux/amd64`.
- `run-dev.sh` - dev loop: pins the router, builds the image from the working tree, submits the Job, waits, prints capacity.json.
- Subdirectory contract: [protogen](./protogen/EGM.md). Also `tests/` (pytest suite, no contract).

## Invariants

- The controller varies per-worker `requests_per_second`, so `initial_value = initial_total_rps / client_concurrency`; the traffic template must not set `duration` and `open_loop` is forced true by the controller (`spec.py:build_spec_dict` docstring).
- All five default failure predicates are overridden to huge bounds so measuring periods run to completion and overload scores as degraded success-rate (`spec.py:PERMISSIVE_FAILURE_PREDICATES`).
- Send-rate is the open-loop backstop: skipped requests are neither failures nor latency samples, so without it the search ramps unboundedly (`spec.py:build_spec_dict` threshold comment).
- Client concurrency is decoupled from `--envoy-cpu` so the client never binds before the router (`runner.py:parse_args` comment).
- `create_and_warm` appends each name to `created` at creation time so the runner's `finally` block can clean up a partially-warmed fleet (`actors.py:create_and_warm`, `runner.py:main`).
- The actors stay resumed for the whole session and are measured warm only, so the ladder never measures a cold start (`actors.py:_warm_actor`).
- Textproto output is rendered without a descriptor pool for `MessageToString` Any expansion: expanded-form Any is not parseable by the client for plugin types it does not link (`spec.py:spec_dict_to_textproto` docstring).

## Side Effects

- Calls ateapi (`CreateAtespace`, `CreateActor`, `ResumeActor`, `DeleteActor`) on the `ingress-benchmark` atespace by default (`actors.py:ATESPACE`).
- Reads `/run/servicedns-ca/ca.crt` and the pod-certificate bundle (or the projected SA token) for authentication (`actors.py`, `spec.py:CA_FILE`, `spec.py:CRED_BUNDLE`).
- Runs local subprocesses `nighthawk_service` and `nighthawk_adaptive_load_client`, and HTTP POSTs `/ping` through the router.
- Writes a `/tmp/<timestamp>-nighthawk-runner` work dir and uploads status.json, logs.txt, spec.textproto, output.textproto, results.json, stats.jsonl, capacity.json to GCS or local disk (`runner.py:main`, `runner.py:upload_to_gcs`).

## Verification

- Run: `hack/verify/benchmarking-python.sh` (runs the `tests/` suite from this directory).
