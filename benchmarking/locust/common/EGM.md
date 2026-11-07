# common

Shared runtime for the locust benchmark users: locust custom flags, one flag
per concern and each registered by the module that owns it; the mTLS channel
to ateapi; tracing, gRPC-gevent setup, and Prometheus metrics. The generated
Python gRPC clients (`ateapi_pb2`, `glutton_pb2`) also land here at image
build time via `../codegen/generate.sh`; they are not checked in.

## Interface

- `resume_mode.py:add_resume_mode_arguments` - `--resume-mode` `explicit|implicit` (default `explicit`).
- `lifecycle_mode.py:add_lifecycle_mode_arguments` - `--lifecycle-mode` `sync|async|none` (default `async`).
- `durdir_config.py:add_durdir_arguments` - `--durdir-file-size-bytes` (default 8 MiB), `--durdir-read-mode` `data|digest`, `--durdir-template`.
- `memload_config.py:add_memload_arguments` - `--mem-target`, `--mem-churn`, `--mem-read` (all default empty = disabled; churn/read require `--mem-target`).
- `cpuload_config.py:add_cpuload_arguments` - `--cpu-cores`, `--cpu-duty-cycle` (boomer CPU draw inside the actor).
- `ping_config.py:add_ping_arguments` - `--max-pings-per-wake` (default 1).
- `rate_config.py:add_rate_arguments` - `--target-rps` (0 disables; a ladder step with `@target_rps` changes it mid-run).
- `sweperf_config.py:add_sweperf_arguments` - `--sweperf-template`, `--sweperf-total-steps`, `--sweperf-num-cycles`, `--sweperf-poll-interval-ms`.
- `spawn_config.py:add_spawn_arguments` - `--total-actors`, `--spawn-concurrency`, `--actor-deadline` (0 defers to boomer-worker's own defaults).
- `agentsession_config.py:add_agentsession_arguments` - `--agentsession-script`, `--agentsession-script-file`, `--agentsession-think-scale`, `--agentsession-phase-in`, `--agentsession-resume-retry-budget`, `--agentsession-snapshot-steps`, `--agentsession-awake`.
- `wait_time.py:init_wait_time` / `wait_time.py:dynamic_wait_time` - `--min/--max-wait-time` and `--min/--max-live-time` flags, and the per-task wait draw from them.
- `trace.py:init_tracing` / `trace.py:set_trace_probability` / `trace.py:get_tracer` - `--trace-probability`, one process-global OTLP `TracerProvider` (service name `substrate-locust`) with an `UpdatableSampler` the ladder can retune mid-run.
- `grpc_tracing.py:traced_grpc` / `grpc_tracing.py:TracedMetadata` - context manager wrapping a unary gRPC call with a span, W3C context injection, and the locust `events.request` fire; reports the server's `x-server-elapsed-us` trailer when the caller assigns `metadata.call`.
- `grpc_setup.py:init_grpc_gevent` - routes gRPC's poller through the gevent hub.
- `ateapi_channel.py:ateapi_channel` / `ateapi_channel.py:_split_cred_bundle` - mTLS channel to ateapi authenticating as this pod, from the pod-certificate bundle.
- `atespace.py:ensure_atespace` / `atespace.py:ATESPACE` - idempotent `CreateAtespace` on the shared `"benchmark"` atespace, wrapped in `traced_grpc`.
- `metrics.py:init_metrics` / `metrics.py:on_request` / `metrics.py:update_user_count` - Prometheus-backed OTel meters `locust_requests_total`, `locust_request_duration_milliseconds`, `locust_users`, fed from locust's request event.
- `boomer_config.py:_FLAGS` / `boomer_config.py:build_config_json` / `boomer_config.py:serve_config_headless` / `boomer_config.py:init_boomer_config` - the boomer-tunable flag set, the `--config-json` payload for headless runs, the loopback `/boomer-config` server, and the web-UI `/boomer-config` route.

## Invariants

- Flag registration lives in the module that owns each flag; `boomer_config.py` only ties them together (`boomer_config.py` module docstring).
- Every `init_*` helper (`wait_time`, `trace`, `grpc_setup`, `metrics`, `boomer_config`) is guarded by a module-level `_initialized` flag, because several test modules load into one locust process (`grpc_setup.py:init_grpc_gevent`, `trace.py:init_tracing`).
- `init_grpc_gevent` must run after locust's gevent monkey-patching and before any gRPC channel is created; upstream `init_gevent()` is not idempotent, so the guard is mandatory (`grpc_setup.py:init_grpc_gevent`).
- A flag added to `boomer_config.py:_FLAGS` needs a matching field in the boomer user class that consumes the payload (`boomer_config.py` module docstring).
- `ensure_atespace` treats `ALREADY_EXISTS` as success so concurrent users can race without coordinating (`atespace.py:ensure_atespace`).
- All locust files share one process-global TracerProvider, so the service name is fixed at `substrate-locust` (`trace.py:SERVICE_NAME_VALUE`).

## Side Effects

- `ateapi_channel` reads `/run/servicedns-ca/ca.crt` and `/run/podidentity.podcert.ate.dev/credential-bundle.pem`, projected by the locust Deployment (`ateapi_channel.py:CA_FILE`, `ateapi_channel.py:CRED_BUNDLE`).
- `metrics.py:init_metrics` starts a Prometheus HTTP server on port 8000; `boomer_config.py:serve_config_headless` binds `/boomer-config` on 127.0.0.1 in a daemon thread.
- `trace.py:init_tracing` registers an OTLP gRPC span exporter; `traced_grpc` fires locust `events.request` and spawns a gevent greenlet for its log line.

## Verification

- Run: `hack/verify/benchmarking-python.sh` (pytest suites import these modules by bare name).
