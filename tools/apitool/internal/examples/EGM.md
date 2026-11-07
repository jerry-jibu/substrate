# examples

## Overview

Hand-written YAML examples of each resource, shown beside the resource's
fields on the doc site. `grpc/` holds one file per gRPC resource in
protojson-as-YAML form (as ate-api-server returns it, server-set fields
included); `k8s/` holds one file per Kubernetes kind as users apply it.

## Interface

- `examples.go:GRPCResources` - embedded `grpc/*.yaml` keyed by message name (e.g. `Actor.yaml` under `Actor`).
- `examples.go:K8sResources` - embedded `k8s/*.yaml` keyed by kind (e.g. `WorkerPool.yaml` under `WorkerPool`).
- Embed source: `examples.go:files` (`//go:embed grpc/*.yaml k8s/*.yaml`).
- Subdirectories: none.

## Invariants

- Tests require every resource to have an example, each example to parse as its message, and each k8s example to validate against its CRD schema (no CEL rules or metadata checks) (`examples.go` package comment).

## Side Effects

- ["None"] (embedded FS reads only)

## Verification

- Run: `cd tools/apitool && go test ./...`
