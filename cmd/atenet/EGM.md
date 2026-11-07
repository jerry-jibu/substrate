# atenet

## Overview

The `atenet` combined networking daemon binary. A thin `main.go` over
`cmd/atenet/internal`, which registers two cobra subcommands: `router`
(ingress/egress routing with an extproc authorization path) and `sdsmint`
(gateway certificate minting). Design details are in `README.md` in this
directory.

## Interface

- `main.go:main` - calls `internal.Execute()`; the binary is only an entry point.
- `internal/root.go:rootCmd` - cobra root `atenet`, version from `internal/version.String()`, persistent flag `--preview` (gates validated by `internal/preview.Init` in `PersistentPreRunE`).
- `internal/root.go:Execute` - runs the root command; prints the error and exits 1 on failure.
- Subcommands registered in `internal/root.go:init()`: `router.NewRouterCmd()` (`cmd/atenet/internal/router`) and `sdsmint.NewSdsmintCmd()` (`cmd/atenet/internal/sdsmint`).

## Invariants

- Preview flags are validated before any subcommand runs (`internal/root.go:rootCmd` `PersistentPreRunE`).

## Side Effects

- Serving gateways open network listeners; the implementations live under `cmd/atenet/internal/router` and `cmd/atenet/internal/sdsmint` (see their contracts below).

## Subdirectories & System Boundaries

- internal/: [./internal/EGM.md](./internal/EGM.md) - the daemon implementation. Child contracts:
  - [./internal/sdsmint/EGM.md](./internal/sdsmint/EGM.md) - sdsmint subcommand.
  - [./internal/router/EGM.md](./internal/router/EGM.md) - the Envoy control plane and ext_proc server.
  - [./internal/router/ingress/EGM.md](./internal/router/ingress/EGM.md) - ingress routing.
  - [./internal/router/extproc/EGM.md](./internal/router/extproc/EGM.md) - extproc authorization.
  - [./internal/router/egress/EGM.md](./internal/router/egress/EGM.md) - egress policy enforcement.
  - [./internal/router/egress/actorjwt/EGM.md](./internal/router/egress/actorjwt/EGM.md) - actor JWT handling for egress.

## Verification

- Run: `go test ./cmd/atenet/...`
