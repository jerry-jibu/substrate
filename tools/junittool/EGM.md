# junittool

## Overview

Standalone Go tool (own go.mod, no dependencies — it runs on the critical
path of every CI job) that records the JUnit files a CI run will write,
then checks they reported tests. An exit code cannot distinguish "all
tests passed" from "no test ran": `go test ./...` passes e2e suites with no
cluster, and a package filter that stops matching reports success having
run nothing.

## Interface

- `main.go:main` - three subcommands plus help; unknown subcommand exits 2 (`main.go:usage`).
  - `register [-manifest FILE] [-reject-duplicate] <junit.xml>` - appends the path to the manifest (`manifest.go:register`, default manifest beside the entry via `manifest.go:defaultManifest`; `manifest.go:errDuplicateEntry` when `-reject-duplicate`, default true when `main.go:inCI` reads `CI=true`).
  - `verify (-manifest FILE | paths...) [-min N]` - every recorded file must hold at least `min` (default 1) test cases (`junit.go:evaluate`, `junit.go:parse`, dir expansion in `junit.go:expand`); an empty manifest fails with "nothing ran".
  - `report [-summary FILE] [-annotate] [-root DIR] <test.json|dir>...` - lists each failed test with its `t.Error`/`t.Fatal` output from `go test -json` files (`report.go:parseRun`, `report.go:event`); appends a markdown summary capped at `report.go:summaryBudget` (900 KiB, GitHub's 1 MiB limit) and emits at most `report.go:maxAnnotations` GitHub annotations; defaults wired to `GITHUB_STEP_SUMMARY`/`GITHUB_ACTIONS`/`GITHUB_WORKSPACE`.
- Subdirectories: none.

## Invariants

- Registering before the run makes the expected set come from the runs themselves, and distinguishes a run that died part-way from one that never existed (`main.go` package comment).
- `report` only reports; the test steps and `verify` decide whether the job fails (`main.go:runReport` doc).
- `go test -json` inputs for `report` must come from runs without `-v`, which replaces the `-test.v=test2json` marker (`main.go:runReport` NOTE).
- Locally a repeat registration is ordinary (permissive), in CI it errors (`main.go:inCI`, mirroring `cmd/ateapi/internal/store/dockerenv.Required`).

## Side Effects

- Appends to and reads the manifest file; reads JUnit XML and go test JSON files; appends a summary file; prints GitHub annotations.

## Verification

- Run: `cd tools/junittool && go test ./...`
