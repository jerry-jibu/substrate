# apitool

## Overview

Standalone Go tool (own go.mod, run as `go -C tools/apitool run .`, `hack/site.sh:46`): validates
`api/ateapi/v1/ateapi.proto` against Substrate's API style guidelines with
a checked-in exemptions file, and generates the API reference pages for
the project website.

## Interface

- `main.go:main` - entry point; runs `cmd.Execute()` and exits 1 on error.
- `go.mod` - separate module from the repo root, so the tool's dependencies do not burden the main module.
- Subdirectories: [./internal/EGM.md](./internal/EGM.md) - commands, model, lint, exemptions, examples, renderer.

## Invariants

- The validated and documented surface is the `Control` service; `WorkerService` stays internal (`internal/validate/validate.go:serviceName`, `internal/cmd/generate_docs.go:documentedService`).
- Exemptions are explicit and sorted; a stale or unsorted file fails validation (`internal/exemption/exemption.go:CheckSorted`).

## Side Effects

- Writes the generated website reference tree and the exemptions file when asked (`--update`, `generate-docs --out-dir`).

## Verification

- Run: `cd tools/apitool && go test ./...`
