# ate-setup/internal/demos/jupyter

## Overview

Installs the Jupyter demo, which runs an unmodified `quay.io/jupyter/base-notebook`
image as an actor that checkpoints on suspend and resumes when the notebook is
next opened (`jupyter.go` package doc).

## Interface

- `init` (`jupyter.go`) - registers a plain `demos.Substrate` as `demo-jupyter`, pool manifest `demos/jupyter/jupyter.yaml.tmpl`, template `jupyter` in atespace `ate-demo-jupyter`.

## Invariants

- ["None"]

## Side Effects

- Deploys the `ate-demo-jupyter` namespace, WorkerPool, and ActorTemplate through `demos.Substrate.Deploy` (`jupyter.go` `init` wiring).

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/jupyter/...`
