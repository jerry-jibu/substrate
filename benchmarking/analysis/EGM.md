# analysis

Turns the node-side logs a benchmark run leaves behind into a suspend/resume
timing report: a collector script that dumps the pods' logs, and the Python
reader that aggregates the phase-breakdown log records both layers emit.

## Interface

- `collect_logs.sh` - dumps every atelet pod (`app=atelet`, namespace `ate-system`) and every worker pod (`ate.dev/worker-pool`, namespace `benchmark-workloads`) log for a time window; flags `--dest` (required), `--since` / `--since-time`, `--namespace`, `--worker-namespace`.
- `phase_report.py:main` - CLI: `phase_report.py run-logs/*.log [--csv DEST_DIR] [--slowest N]`; aggregates the two joinable record kinds ("Restore timing breakdown", "Checkpoint timing breakdown") written by both atelet and ateworker.
- `phase_report.py:parse_lines` / `phase_report.py:parse_files` - parse kubectl JSON-line dumps, Cloud Logging exports (a JSON array with records under `jsonPayload`), or stdin (`-`); also the entry point for a caller holding log text, such as the locust runner reading pods through the Kubernetes API.
- `phase_report.py:pair_records` / `phase_report.py:join_layers` - match each atelet record to the ateworker record of the same actor and operation inside the atelet `worker_*` phase window, and derive each record's unattributed residual.
- `phase_report.py:report_phases` / `phase_report.py:report_waterfalls` - per-phase percentiles grouped by (layer, op, class, kind, scope, phase), and the slowest operations as nested waterfalls with the RPC/queueing gap.
- `phase_report.py:stats_rows` - the percentile rows reshaped into the benchmark runner's `stats.jsonl` form (`benchmarking/locust/runner.py`).
- `phase_report.py:write_csv` - writes `phase_percentiles.csv` into a destination directory.
- `test_phase_report.py` - unit tests for the parser (not read for this contract).

## Invariants

- Duration-key prefixes are pinned to the emitters: `ate.actor.restore.duration.*`, `ate.actor.checkpoint.duration.*` (atelet) and `ateworker.actor.restore.duration.*`, `ateworker.actor.checkpoint.duration.*` (ateworker), source of truth `cmd/atelet/metrics.go` and `internal/ateworker/phaselog/phaselog.go` (`phase_report.py:BREAKDOWN_PREFIXES`).
- Feed one source per run, kubectl dumps or a Cloud Logging export, not both, or every record counts twice (`phase_report.py` module docstring).
- Pairing matches by actor and op inside a window with `JOIN_SLACK_S` (1.0s) slack, consumes each ateworker record once, and a paired ateworker record inherits `kind` and `sandbox_class` from the atelet record (`phase_report.py:pair_records`).
- The microVM checkpoint captures (`snapshot`, `durable_dir`, `rootfs_upper`) run concurrently on the paused guest, so the paused window costs their max, not their sum; gVisor's phases are sequential (`phase_report.py:CONCURRENT_CAPTURES`, `phase_report.py:unattributed`).
- A negative unattributed residual is a bug in the emitter or the formula, and is printed unclamped so it shows (`phase_report.py:unattributed`).
- A non-numeric duration value costs only that phase, not the run's report (`phase_report.py:parse_line`).
- `collect_logs.sh` warns and moves on when one pod cannot be read, so one bad pod does not cost the others (`collect_logs.sh:collect`); it sources `.ate-dev-env.sh` from the repo root unless `NO_DEV_ENV` is set, and respects `KUBECTL_CONTEXT`.

## Side Effects

- `collect_logs.sh` runs `kubectl get pods` and `kubectl logs` (including `--previous` for restarted containers) and writes one `.log` file per pod under `--dest` (`collect_logs.sh:collect`).
- `phase_report.py` reads log files or stdin and, with `--csv`, writes `phase_percentiles.csv` under the destination directory (`phase_report.py:write_csv`).

## Verification

- `python3 benchmarking/analysis/phase_report.py --help` runs the CLI self-check; no test runner is configured in this directory.
