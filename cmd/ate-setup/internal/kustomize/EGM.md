# ate-setup/internal/kustomize

## Overview

Renders the overlays under `manifests/ate-install`, standing in for the shell
installer's `kubectl kustomize` calls (`kustomize.go` package doc).

## Interface

- `Build` (`kustomize.go`) - render an overlay directory to a multi-document YAML manifest; load restrictions are disabled because the kind overlays need to reach outside their root.
- `Compose` / `ComposeBytes` (`kustomize.go`) - render a manifest file, directory, or in-memory manifest with extra patches applied, through a throwaway kustomization in a temporary directory.

## Invariants

- Load restrictions stay disabled to match the shell scripts: the kind overlays load from outside their directory (`kustomize.go` `Build`).
- Composition is a throwaway kustomization in a temporary directory, not a mutation of the source manifests (`kustomize.go` `Compose`).

## Side Effects

- Creates and removes a temporary directory per Compose call (`kustomize.go` `Compose`).

## Verification

- Run: `go test ./cmd/ate-setup/internal/kustomize/...`
