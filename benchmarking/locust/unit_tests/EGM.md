# unit_tests

The pytest suite for the locust harness. `hack/verify/benchmarking-python.sh`
runs it from `benchmarking/locust` with `(cd benchmarking/locust && python -m
pytest -q unit_tests)`, because the modules under test are imported by bare
name.

## Interface

- `test_boomer_config.py` - tests for `common/boomer_config.py` (module docstring).
- `test_cluster_facts.py` - tests for `cluster_facts.py` (module docstring).
- `test_ladder_shape.py` - tests for `shapes/ladder_shape.py` (module docstring).
- `test_ladder_steps.py` - tests for `ladder_steps.py` (module docstring).
- `test_phase_breakdown.py` - tests for `phase_breakdown.py` (module docstring).
- `test_server_telemetry.py` - tests for `server_telemetry.py` (module docstring).
- `test_turnover_window.py` - tests for `turnover_window.py` (module docstring).
- Each test file inserts the parent directory on `sys.path` so `common` and the top-level modules import (for example `test_boomer_config.py` `sys.path.insert`).

## Invariants

- `conftest.py` imports `locust` before any test module: locust calls `gevent.monkey.patch_all()` on import, and it must run before a module imports `ssl` through the kubernetes client, or a later patch makes `SSLContext` recurse (`conftest.py` comment).

## Side Effects

- ["None"]

## Verification

- Run: `hack/verify/benchmarking-python.sh` (stubs `docker`, `gcloud`, `kubectl` and sets `KUBECONFIG=/nonexistent` so a test that escapes its mocks cannot reach a cluster or registry).
