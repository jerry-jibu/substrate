# ate-setup/internal/demos/egress

## Overview

Installs the egress demo, which exercises egress policy enforcement through
atenet. Its actors project the egress gateway trust bundle: a golden snapshot
only exists once an actor starts, and an actor whose trust bundle does not
resolve never does, so a golden wait timeout is the symptom of a missing egress
gateway CA (`egress.go` package doc).

## Interface

- `init` (`egress.go`) - registers a plain `demos.Substrate` as `demo-egress`, pool manifest `demos/egress/egress.yaml.tmpl`, template `egress` in atespace `ate-demo-egress`.

## Invariants

- ["None"]

## Side Effects

- Deploys the `ate-demo-egress` namespace, WorkerPool, and ActorTemplate, and blocks on the golden snapshot, through `demos.Substrate.Deploy` (`egress.go` `init` wiring).

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/egress/...`
