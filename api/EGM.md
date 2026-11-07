# api Blueprint

## System Overview

Public APIs of Agent Substrate, one directory per API and version. Two kinds
of API live here: gRPC APIs defined as protobuf (the `Control` plane API, the
credential-provider plugin API, and the object-store plugin API) and
Kubernetes CRD types under `infra/k8s` (the `ate.dev` group). All packages
belong to the root Go module, `go.ate.dev/substrate`.

## Subdirectories & System Boundaries

- ateapi: [./ateapi/EGM.md](./ateapi/EGM.md) - the `Control` and
  `WorkerService` gRPC APIs served by `ate-api-server`.
- credprovider: [./credprovider/EGM.md](./credprovider/EGM.md) - the
  `CredentialProvider` plugin API for fetching external credentials.
- infra: [./infra/EGM.md](./infra/EGM.md) - Kubernetes CRD types;
  `infra/k8s/ate/v1` defines `WorkerPool`, `SandboxConfig`, `CSIDriverConfig`.
- objectstore: [./objectstore/EGM.md](./objectstore/EGM.md) - the
  `NodeProvider`, `ControlProvider`, and `AssetProvider` object-store plugin
  APIs.

## Global Invariants

- Protos are the source of truth for the gRPC APIs; the `*.pb.go` files are
  generated and must be regenerated rather than hand-edited (see
  `api/ateapi/v1/ateapi.proto`, `api/credprovider/v1/credprovider.proto`,
  `api/objectstore/v1/objectstore.proto`).
- Proto validation is expressed with `+k8s:*` annotations that codegen turns
  into enforceable bounds (`api/ateapi/v1/ateapi.proto` field comments).
- Kubernetes types register themselves into the `ate.dev/v1` scheme on
  import (`api/infra/k8s/ate/v1/groupversion_info.go:27`).

## Global Side Effects

- CRD and client codegen triggered from
  `api/infra/k8s/ate/v1/gen.go:17`-`:20` writes generated artifacts outside
  this tree (`manifests/ate-install/generated/`, `pkg/client/`).

## System Verification

- Run: `go test ./api/...`
