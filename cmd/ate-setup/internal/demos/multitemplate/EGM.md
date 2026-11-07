# ate-setup/internal/demos/multitemplate

## Overview

Installs the multi-template demo, where two ActorTemplates in two different
atespaces share a single WorkerPool (`multitemplate.go` package doc).

## Interface

- `init` (`multitemplate.go`) - registers a plain `demos.Substrate` as `demo-multi-template`, pool manifest `demos/multi-template/multi-template.yaml.tmpl`, deployment `shared-pool` in `ate-demo-multi-template-pool`, and templates `counter` and `fspersist` in their own atespaces.

## Invariants

- ["None"]

## Side Effects

- Deploys the pool namespace and two template atespaces through `demos.Substrate.Deploy` (`multitemplate.go` `init` wiring).

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/multitemplate/...`
