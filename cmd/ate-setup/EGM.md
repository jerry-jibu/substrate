# ate-setup

## Overview

Command `ate-setup` installs and tears down Agent Substrate on a Kubernetes
cluster (`main.go` package doc). The binary is a thin entry point: `main` calls
`cmd.Execute()` and everything else lives under `internal/`.

## Interface

- `main` (`main.go`) - calls `cmd.Execute()`, the whole command tree.
- [internal](./internal/EGM.md) - the command tree, configuration, steps, demos, and build/cluster helpers.

## Invariants

- ["None"] (the entry point adds no logic of its own; see `./internal/cmd/EGM.md` for the command invariants).

## Side Effects

- ["None"] at this level; the subcommands under `./internal` mutate the cluster, the container registry, and the local filesystem.

## Verification

- Run: `go test ./cmd/ate-setup/...`
