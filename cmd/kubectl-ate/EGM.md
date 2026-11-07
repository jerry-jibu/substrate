# kubectl-ate

## Overview
The `kubectl ate` plugin binary for managing Agent Substrate environments: actor and worker lifecycles, atespaces, templates, tags, egress and access policies, admin key pools, and a local proxy to a resource.

## Interface
- `main.go:main` - thin entry point calling `cmd.Execute()`.
- [internal/cmd](./internal/EGM.md) - the cobra command tree and persistent flags.
- [internal/printer](./internal/EGM.md) - output rendering.

## Invariants
- Delegates entirely to `internal/cmd`; `main.go` contains no logic of its own.

## Side Effects
- See [internal/cmd](./internal/EGM.md): gRPC calls to ate-api, Kubernetes port-forwarding, local proxy listeners.

## Verification
- Run: `go test ./cmd/kubectl-ate/...`
