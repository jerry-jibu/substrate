# benchmark-egress

## Overview

The workload of the egress scale test, which holds many CONNECT tunnels
open through the egress gateway at a fixed pace and measures what they
cost. One binary, one cobra subcommand per program: `actor` (opens the
tunnels and counts what happened) and `target` (the endpoint every tunnel
leads to). The full test design, topology, and the driver contract live in
`README.md` in this directory.

## Interface

- `main.go:main` - builds the `benchmark-egress` cobra root with `SilenceUsage`, `SilenceErrors`, and default completion disabled; registers `actor.Command()` and `target.Command()` (`cmd/benchmarking/egress/main.go:main`).
- Context: `signal.NotifyContext` on SIGINT/SIGTERM; errors print once to stderr and exit 1 (`main.go:main`).
- Subdirectories: [./internal/EGM.md](./internal/EGM.md) - subcommand implementations and target certificate.

## Invariants

- The actor's loop starts only on `POST /start`, never at boot, because actors restore from a golden snapshot taken before any egress policy exists (`../egress/internal/cmd/actor/actor.go:Command` doc comment).
- Every actor answer carries `Benchmark-Egress-Protocol: 2`; drivers must refuse version 1 images (`../egress/internal/cmd/actor/api.go:ProtocolHeader`).

## Side Effects

- The actor serves its control HTTP server on port 80 inside the sandbox (`../egress/internal/cmd/actor/actor.go:runActor`).
- The target opens TCP listeners per `--listen`, `--listen-ports`, and the TLS equivalents (`../egress/internal/cmd/target/target.go:runTarget`).

## Verification

- Run: `go test ./cmd/benchmarking/egress/...`
