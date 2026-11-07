# version

## Overview

Build-time identity for substrate binaries. Values are set via
`-ldflags -X`; unset values fall back to `runtime/debug.ReadBuildInfo()`
so plain `go build` / `go install` still produce meaningful output.

## Interface

- `version.go:Version`/`Commit`/`BuildDate` - build-time vars; `init()` fills Commit/BuildDate from `vcs.revision`/`vcs.time`/`vcs.modified` (append `-dirty`) and Version from the module path and version.
- `version.go:String` - one-line summary `"<version> commit=<c> built=<d> <goos>/<goarch>"`, shaped to follow cobra's `version` prefix.
- Subdirectories: none.

## Invariants

- ldflags-provided values win; the build-info fallback only fills `unknown` values (`version.go:init`).

## Side Effects

- ["None"]

## Verification

- Run: `go test ./internal/version/...`
