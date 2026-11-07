# ate-setup/internal/demos/sandbox

## Overview

Installs the sandbox demo: an ActorTemplate driven on demand by the sandbox
client rather than by a long-lived workload (`sandbox.go` package doc).

## Interface

- `init` (`sandbox.go`) - registers a plain `demos.Substrate` as `demo-sandbox`, pool manifest `demos/sandbox/sandbox.yaml.tmpl`, deployment `sandbox-workerpool`, template `sandbox-template` in atespace `ate-demo-sandbox`.

## Invariants

- ["None"]

## Side Effects

- Deploys the `ate-demo-sandbox` namespace, WorkerPool, and ActorTemplate through `demos.Substrate.Deploy` (`sandbox.go` `init` wiring).

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/sandbox/...`
