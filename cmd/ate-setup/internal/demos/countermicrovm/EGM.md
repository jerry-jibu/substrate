# ate-setup/internal/demos/countermicrovm

## Overview

Installs the micro-VM variant of the counter demo. It has no external-volume
option: the CSI path is covered by demo-counter (`countermicrovm.go` package
doc).

## Interface

- `init` (`countermicrovm.go`) - registers a plain `demos.Substrate` as `demo-counter-microvm`, pool manifest `demos/counter/counter-microvm.yaml.tmpl`, template `counter-microvm` in atespace `ate-demo-counter-microvm`, with `GoldenTimeout: demos.MicroVMGoldenTimeout`.

## Invariants

- Golden snapshot waits use `demos.MicroVMGoldenTimeout` (600s), because a micro-VM golden is a cloud-hypervisor cold boot plus a checkpoint (`countermicrovm.go` `init`, `../substrate.go` `MicroVMGoldenTimeout`).

## Side Effects

- Deploys the `ate-demo-counter-microvm` namespace, WorkerPool, and ActorTemplate through `demos.Substrate.Deploy` (`countermicrovm.go` `init` wiring).

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/countermicrovm/...`
