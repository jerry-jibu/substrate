# ate-setup/internal/demos/egressmicrovm

## Overview

Installs the micro-VM variant of the egress demo. It is a demo of its own
rather than a flag on demo-egress so that it appears in help and in the
`delete all` sweep, and so both can be installed side by side: the networking
e2e suite runs against whichever the sandbox class under test selects
(`egressmicrovm.go` package doc).

## Interface

- `init` (`egressmicrovm.go`) - registers a plain `demos.Substrate` as `demo-egress-microvm`, pool manifest `demos/egress/egress-microvm.yaml.tmpl`, template `egress-microvm` in atespace `ate-demo-egress-microvm`, with `GoldenTimeout: demos.MicroVMGoldenTimeout`.

## Invariants

- Golden snapshot waits use `demos.MicroVMGoldenTimeout` (`egressmicrovm.go` `init`).

## Side Effects

- Deploys the `ate-demo-egress-microvm` namespace, WorkerPool, and ActorTemplate through `demos.Substrate.Deploy` (`egressmicrovm.go` `init` wiring).

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/egressmicrovm/...`
