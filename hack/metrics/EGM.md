# metrics

One-off analysis helper for CI health, next to the metric registry under
`docs/metrics/`. Not part of `hack/verify-all.sh`.

## Interface

- `ci-failure-analysis.py:classify` - maps a failed job log to one root-cause category: `no_free_workers`, `e2e_timeout`, `gcs_access`, `named_test_fail`, `verify_fail`, `license_check`, `log_fetch_failed`, `unknown`.
- `ci-failure-analysis.py:analyse` - fetches up to `--fetch-limit` workflow runs, samples `--sample` failed ones, and classifies each through the `gh` CLI.
- `ci-failure-analysis.py:print_report` / `ci-failure-analysis.py:main` - the category breakdown plus flaky-test surfacing; `--save` writes raw JSON for later diffing, `--repo` selects the repository (default `agent-substrate/substrate`).

## Invariants

- Patterns are ordered and first-match-wins: infrastructure patterns precede `named_test_fail` so a run that dies on "no worker has room" is not attributed to the Go test that printed `--- FAIL:` before it (`ci-failure-analysis.py` classification comment).
- `verify_fail` matches only explicit verify-script failure output; its older broad substring swallowed almost every verify job (`ci-failure-analysis.py` classification comment).
- Generic timeout strings are deliberately excluded from `e2e_timeout` because they appear in ordinary unit-test output (`ci-failure-analysis.py` classification comment).

## Side Effects

- Calls `gh run list`/`gh run view` (read-only) and optionally writes a JSON file (`ci-failure-analysis.py:_gh_json`, `--save`).

## Verification

- ["None"] (no test or verifier runs this script)
