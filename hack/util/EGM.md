# util

Shared helpers for the other hack/ scripts: Python venv management, the
boilerplate license-header checker behind `hack/verify/boilerplate.sh`, and
the GitHub contributor lister used by `hack/release/draft-release.sh`.

## Interface

- `venv.sh:venv_is_usable` / `venv.sh:ensure_venv` / `venv.sh:venv_sync_requirements` - sourced by scripts that need a Python venv (benchmarking suites, proto codegen); `ensure_venv` recreates with `--clear` when the venv is unusable.
- `verify-boilerplate.py:verify_file` / `verify-boilerplate.py:main` - checks tracked files for the `Copyright 2026..N The Agent Substrate Authors` line plus the Apache-2.0 header lines in order, skipping non-source extensions, `third_party`, `vendor`, and `_LICENSES/`.
- `get-contributors.sh` - lists unique GitHub logins for a revision range (default `HEAD`), resolving each commit's authors (including `Co-authored-by` trailers) through the GitHub GraphQL API; `ORG`/`REPO` env vars select the repository.

## Invariants

- `venv_is_usable` executes the venv's interpreter, because an interrupted create or an interpreter upgrade leaves a dangling `bin/python3` (`venv.sh:venv_is_usable` comment).
- `ensure_venv` uses `--clear`: plain `python3 -m venv` and `--upgrade` both leave the dangling interpreter in place (`venv.sh:ensure_venv` comment).
- The copyright year must be within `2026..today` and the nine license lines must appear in order, resilient to comment-marker and whitespace variation (`verify-boilerplate.py:match_copyright`, `verify-boilerplate.py:verify_file`).
- `get-contributors.sh` fails on an invalid GitHub response rather than silently dropping a commit (`get-contributors.sh` jq -e check).

## Side Effects

- `verify-boilerplate.py:main` runs `git ls-files --modified --cached`; `get-contributors.sh` runs `git log` and `gh api graphql` and writes a temp dir (`mktemp -d`, removed on exit).

## Verification

- Run: `hack/verify/shellcheck.sh` for the shell helpers; `hack/verify/boilerplate.sh` invokes `verify-boilerplate.py` (the checker does not check itself).
