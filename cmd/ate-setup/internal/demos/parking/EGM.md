# ate-setup/internal/demos/parking

## Overview

Installs the parking demo, which parks and unparks actors on a small WorkerPool
(`parking.go` package doc).

## Interface

- `init` (`parking.go`) - registers a plain `demos.Substrate` as `demo-parking`, pool manifest `demos/parking/parking.yaml.tmpl`, template `parking` in atespace `ate-demo-parking`.

## Invariants

- ["None"]

## Side Effects

- Deploys the `ate-demo-parking` namespace, WorkerPool, and ActorTemplate through `demos.Substrate.Deploy` (`parking.go` `init` wiring).

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/parking/...`
