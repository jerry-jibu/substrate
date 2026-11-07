# verify

## Overview

Standalone verification scripts, one check per script, invoked locally and
from CI. Each script should have a corresponding entry in
`../verify-all.sh` (`README.md` in this directory). All scripts `cd` to the
repo root via `git rev-parse --show-toplevel` and run with
`set -o errexit -o nounset -o pipefail`.

## Interface

- `boilerplate.sh` - runs `hack/util/verify-boilerplate.py` (copyright headers).
- `codegen.sh`, `gofmt.sh`, `golangci-lint.sh`, `go-modules.sh` - generated code, formatting, lint, and module tidiness.
- `proto-fmt.sh`, `proto-breaking.sh`, `python-protos.sh` - proto formatting, breaking-change detection, and generated Python protos.
- `licenses.sh`, `python-licenses.sh` - license checks.
- `metrics.sh` - runs OpenTelemetry Weaver `registry check --registry docs/metrics/registry --future` with a version-gated local binary or the digest-pinned `otel/weaver:v0.25.1` image (`metrics.sh:run_weaver`, `metrics.sh:WEAVER_VERSION`).
- `postgresql-migrations.sh` - migration checks.
- `package-deps.sh`, `pkg-external-import.sh` - package dependency rules.
- `kube-api-linter.sh` - Kubernetes API lint.
- `shellcheck.sh` - shell lint.
- `threats.sh` - threat-model checks.
- `benchmarking-python.sh` - benchmarking Python checks.

## Invariants

- The local `weaver` binary is used only when its version matches `WEAVER_VERSION`; any other version fails the script so a green local run means the same as CI (`metrics.sh:run_weaver`).
- Scripts fail loudly: they exit non-zero on the first failed check and print the failing file or fix (`boilerplate.sh`, `metrics.sh`).

## Side Effects

- `metrics.sh:run_weaver` runs a Docker container (`${DOCKER:-docker}`) mounting the repo read-only when no local binary is used.

## Verification

- Run: `hack/verify-all.sh` (or any single `hack/verify/<script>.sh`)
