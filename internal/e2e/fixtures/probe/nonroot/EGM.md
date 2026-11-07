# nonroot

## Overview

Command `nonroot` is the e2e probe for an image with a non-root USER and a WORKDIR that user owns. It reports the identity, effective capabilities, and working directory the actor's process runs with. It is a separate binary from the probe because ko picks the base image per import path: this one builds on the default nonroot distroless base, whose WORKDIR is `/home/nonroot`.

## Interface

- `main.go:identity` - serves `/identity`, returning `main.go:identityResponse`: `uid`, `gid`, `groups`, `cwd`, `capEff` (hex mask from `/proc/self/status`), and `writeError` for the write checks.
- `main.go:main` - also serves `/healthz` (200) and listens on `:80`.

## Invariants

- The write check tries to create and remove a temp file in `.`, `/tmp`, and `/dev/shm`, and concatenates every failure into `writeError` so an assertion explains itself (`main.go:identity`).
- Read failures (getgroups, getwd, `/proc/self/status`) are reported in `error`, never swallowed (`main.go:identity`).

## Side Effects

- Creates and removes temp files in the working directory, `/tmp`, and `/dev/shm` (`main.go:identity`).
- Binds a listener on port 80 (`main.go:main`).

## Verification

- Run: `go test ./internal/e2e/fixtures/probe/nonroot/...`
