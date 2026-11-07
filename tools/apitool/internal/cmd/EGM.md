# cmd

## Overview

The cobra command tree of `apitool`: `validate` (API shape rules with
exemptions) and `generate-docs` (website API reference).

## Interface

- `root.go:Execute` - runs the `apitool` root command (`root.go:rootCmd`); `main.go` prints the error and exits 1.
- `validate.go:runValidate` - builds the API, runs `validate.All`, matches findings against the exemptions file, reports per-rule pass/fail with ANSI icons honoring `NO_COLOR` (`validate.go:colorEnabled`); `--update` rewrites `exemptions.json` to match current findings (`validate.go:updateExemptions` via `exemption.Save`); stale exemptions fail the run.
- `generate_docs.go:runGenerateDocs` - renders `latest/` from this checkout plus one directory per release with `--releases-since` (`releases.go:readReleases`, read from git); writes `reference.html` and `nav.json` per version (`generate_docs.go:writeVersion`), then `versions.json` (newest first) and `docsite.Assets` files into `--out-dir` (required).
- `generate_docs.go:documentedService` - only the `Control` service is documented; `generate_docs.go:latestVersion` (`latest`) names the checkout build.
- Subdirectories: none.

## Invariants

- `--out-dir` is required for `generate-docs` (`generate_docs.go:MarkFlagRequired`).
- Validation fails on any non-exempted finding and on any stale exemption, telling the operator to run `--update` (`validate.go:runValidate`).

## Side Effects

- Writes `reference.html`, `nav.json`, `versions.json`, and asset files (0644) under `--out-dir`; creates the version directories (`generate_docs.go:writeVersion`, `runGenerateDocs`).
- Reads git tags with `--releases-since` (`releases.go:readReleases`) and rewrites the exemptions file with `--update`.

## Verification

- Run: `cd tools/apitool && go test ./...`
