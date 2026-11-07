# internal

## Overview
Shared code for the atenet binary. `root.go` defines the cobra root command and registers the subcommands, and holds the persistent `--preview` flag wiring.

## Interface
- `root.go:Execute()` runs the `atenet` root command; `root.go:rootCmd` carries `--preview` (`root.go:previewFlags`) and version info.
- Subcommands: `router.NewRouterCmd()` (`./router/EGM.md`) and `sdsmint.NewSdsmintCmd()` (`./sdsmint/EGM.md`).

## Invariants
- `PersistentPreRunE` validates `--preview` gates through `preview.Init` before any subcommand runs (`root.go:rootCmd`).

## Side Effects
- `root.go:Execute` prints errors and exits non-zero on command failure.

## Verification
- Run: `go test ./cmd/atenet/internal/...`
