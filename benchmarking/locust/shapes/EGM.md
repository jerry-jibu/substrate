# shapes

Locust `LoadTestShape` classes that control user count over time. A shape
file is named in the test's `file` after the test itself (for example
`file: /app/tests/glutton.py,/app/shapes/ladder_shape.py`), because locust
ignores `-t` while a shape is present and the shape ends the run.

## Interface

- `spawn_shape.py:SpawnShape.tick` - holds `(1, 1.0)` until the `("TimeToAllReady", "summary")` stats row has at least one request, or `run_time` expires.
- `ladder_shape.py:parse_ladder` - parses a `--ladder` value into `Step` tuples `users, seconds, trace_probability, target_rps`; raises `ValueError` on any step that does not parse or has no duration.
- `ladder_shape.py` flags - `--ladder` (env `LOCUST_LADDER`) and `--ladder-spawn-rate` (default 5 users/s), registered on locust's parser.
- `ladder_shape.py:LadderShape.tick` - holds each step in turn by elapsed-time boundaries, applies the step's trace probability and target rate as the step starts, and returns `None` after the last step.
- `burst_shape.py:BurstShape.tick` - 30 s cycles: ramp to `peak_users = 3` at 1/s over the first 10 s, then hold the currently active user count for the idle phase.

## Invariants

- A step that names no probability or rate leaves it unchanged; `--trace-probability` and `--target-rps` give the values before the first step that names one (`ladder_shape.py` module docstring).
- `SpawnShape.tick` reads the stats table with `.get()`, never indexing `entries`, because indexing creates an empty `StatsEntry` row (`spawn_shape.py:tick`).
- `LadderShape` must not be loaded by itself: without `--ladder`, `tick()` returns `None` at once and locust ends the run; `runner.py` adds the file to `-f` only when the flag is present (`ladder_shape.py:LadderShape` docstring).
- A probability change goes to both consumers: the master's parsed options (what `/boomer-config` serves) and `common.trace.set_trace_probability` for Python user classes (`ladder_shape.py:_set_trace_probability`).

## Side Effects

- `LadderShape.tick` mutates `environment.parsed_options.trace_probability` and `target_rps` in place, which the `/boomer-config` endpoint then serves to boomer workers (`ladder_shape.py:_set_trace_probability`, `ladder_shape.py:_set_target_rps`).

## Verification

- Run: `hack/verify/benchmarking-python.sh` (covers `../unit_tests/test_ladder_shape.py`).
