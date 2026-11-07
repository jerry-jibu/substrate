# site-release-data

## Overview

Root-module Go tool that writes the release metadata the website's
templates read: the version being published, the previous full release,
and the Markdown files added between the two. Output shape is
`site/data/release.json`.

## Interface

- `main.go:main` - flags `--ref` (git ref being published, default `HEAD`) and `--out` (JSON file path, required); exits 2 without `--out`.
- `main.go:releaseData` - `{version, previous, added[]}`.
- `main.go:releaseTag` - matches full `vX.Y.Z` tags only; pre-releases like `v0.3.0-rc.1` do not match.
- `main.go:compute` - resolves the ref, takes the version when the ref is a full release tag, picks the previous release (highest below the current tag, else highest overall), and lists Markdown files added since it.
- Subdirectories: none.

## Invariants

- Only full semver tags count as releases (`main.go:releaseTag`).
- `added` is always a JSON array, never null (`main.go:compute` initializes `Added: []string{}`).

## Side Effects

- Runs git commands in the repo (`main.go:compute` via `main.go:git`) and writes the JSON file (0644, parent dirs created) (`main.go:run`).

## Verification

- Run: `go test ./tools/site-release-data/...` (root module)
