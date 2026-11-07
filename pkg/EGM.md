# pkg

Shared packages intended for external import. Anything here is public
surface: out-of-tree object-store plugin authors, CRD clients, and external
consumers of the protobuf messages import from this module root.

## Subdirectories & Boundaries

- [./assetfetch/EGM.md](./assetfetch/EGM.md) - the checks every object-store plugin applies when serving `AssetProvider.FetchAsset`.
- [./client/EGM.md](./client/EGM.md) - generated Kubernetes client machinery (clientset, informers, listers) for the `ate.dev/v1` CRD types.
- [./objectstorage/EGM.md](./objectstorage/EGM.md) - snapshot object-store I/O shared by atelet and the plugins; GCS and S3 behind one `ObjectStorage` interface.
- [./objectstore/EGM.md](./objectstore/EGM.md) - public packages for out-of-tree object-store providers, including the `actorauthz` credential-exchange authorization library.
- [./postgressetup/EGM.md](./postgressetup/EGM.md) - PostgreSQL identity setup shared by the installers; embeds `setup.sql`.
- [./proto/EGM.md](./proto/EGM.md) - public protobuf packages (`ateconfigpb`).
- [./sandboxclass/EGM.md](./sandboxclass/EGM.md) - the sandbox runtime class constants (`gvisor`, `microvm`) shared by API types and workers.

## Global Invariants

- Packages here are importable by external modules, so they must not import `internal/` packages (repository layout contract, AGENTS.md).
- `pkg/client/` is generated output; regenerate with the generators in `hack/tools/` rather than editing (`./client/EGM.md`).

## System Verification

- Run: `go test ./pkg/...` from the repo root.
