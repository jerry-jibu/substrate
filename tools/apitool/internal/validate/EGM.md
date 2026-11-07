# validate

## Overview

Runs apitool's lint rules against the Substrate API and compares the
result to the checked-in exemptions file. The logic behind both the
`apitool validate` command and its presubmit golden-file test.

## Interface

- `validate.go:API` - builds `model.API` from `ateapi.Source` scoped to `validate.go:serviceName` (`Control`); the one entry point for both the command and the golden test.
- `validate.go:All` - runs every rule in `lint.All`, returning `validate.go:RuleFindings` in rule order.
- `validate.go:AsExemptions` - flattens findings into `exemption.Exemption` identities.
- `validate.go:DefaultExemptionsPath` - `<repo root>/tools/apitool/exemptions.json`, found via `validate.go:repoRoot` (walks up to the go.mod declaring `go.ate.dev/substrate`).
- Subdirectories: none.

## Invariants

- The validated surface is the `Control` service only; `WorkerService` is internal (`validate.go:serviceName` doc).
- An error from a rule means the API is malformed beyond checking and aborts the run, rather than counting as findings (`validate.go:All`).

## Side Effects

- Reads go.mod files up the tree and the exemptions file on disk (`validate.go:repoRoot`).

## Verification

- Run: `cd tools/apitool && go test ./...`
