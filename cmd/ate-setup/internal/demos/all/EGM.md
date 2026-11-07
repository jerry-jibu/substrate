# ate-setup/internal/demos/all

## Overview

Links every bundled demo into the ate-setup binary. Import it for its side
effects; each demo package registers itself with the `demos` package from
`init`. This is the one place that lists them, so a demo added but not linked
shows up here rather than as a silently missing subcommand (`all.go` package
doc).

## Interface

- Blank imports of `claudemultiplex`, `counter`, `countermicrovm`, `egress`, `egressmicrovm`, `jupyter`, `multitemplate`, `parking`, and `sandbox` (`all.go`).

## Invariants

- ["None"]

## Side Effects

- Demo registration happens in each imported package's `init` via `demos.Register` (see `../demos.go`); this package triggers it by being imported.

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/all/...`
