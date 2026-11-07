# ate-setup/internal/ko

## Overview

Builds and publishes images referenced by manifests, replacing the shell
installer's ko invocations (`ko.go` package doc).

## Interface

- `Runner`, `New` (`ko.go`) - invokes the ko binary with a fixed repository root and environment; the ko path is resolved once up front so a missing binary fails early.
- `Resolve` / `ResolveBytes` (`ko.go`) - build and publish the images a manifest (on disk or in memory) references and rewrite it to pullable refs.
- `Build`, `BuildTagged`, `BuildEach` (`ko.go`) - build and publish one import path, a set tagged in one invocation, or one-per-import-path with per-failure retries.
- `Ldflags`, `CommitStamp`, `BuildVersion` (`ko.go`) - version stamping: `-X` flags from the checkout's commit, and `VERSION` else `git describe`.

## Invariants

- A missing ko binary fails at `New`, not mid-deploy (`ko.go` `New`).
- `BuildEach` retries each per-path failure rather than failing the whole batch (`ko.go` `BuildEach`).

## Side Effects

- Runs the ko binary as a subprocess, building and pushing images to the configured repository (`ko.go` `Runner`).
- Reads git state for version stamping (`ko.go` `CommitStamp`, `BuildVersion`).

## Verification

- Run: `go test ./cmd/ate-setup/internal/ko/...`
