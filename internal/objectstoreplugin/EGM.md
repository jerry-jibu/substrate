# objectstoreplugin

## Overview
The object-store plugin binaries' implementation: node, asset, and control plugin servers over GCS or S3 storage, plus the credential exchange that scopes per-actor access ("Package objectstoreplugin", `node.go`). Snapshot files are stored as zstd-compressed objects named `<file>.zstd` under the snapshot URI, except the manifest, which is stored as is.

## Interface
- `node.go:NodePlugin`, built with `node.go:NewNodePlugin(client objectstorage.ObjectStorage, root string, uploadMemory int64, opts ...NodeOption)`, implements the node plugin: `node.go:NodePlugin.FetchSnapshot`, `UploadSnapshot`; `status.go:toStatus` maps plugin errors to gRPC statuses, keeping `codes.NotFound` distinguishable so callers can probe for a snapshot.
- `credentials.go:Credentials` supplies the per-transfer storage client: `credentials.go:Client(ctx, actorJWT, force)`; `credentials.go:Ambient(client)` ignores the JWT (the S3 source). `credentials.go:transfer` replaces a refused credential mid-transfer.
- `asset.go:AssetPlugin`, built with `asset.go:NewAssetPlugin(client, scheme, root, stagingDir string, stagingCapacity int64)`, serves `asset.go:AssetPlugin.FetchAsset`.
- `control.go:ControlPlugin`, built with `control.go:NewControlPlugin(store objectstore.Store)`, serves `control.go:ControlPlugin.CopySnapshot` and `CleanupSnapshot`.
- Plugin APIs: `api/credprovider`-style gRPC services under `pkg/objectstorage` and `internal/objectstore` types.
- Children: `./exchange/EGM.md`, `./exchangeclient/EGM.md`, `./gcsboundary/EGM.md`, `./gcsissuer/EGM.md`, `./objectstoreplugintest/EGM.md`, `./credential/EGM.md`, `./actorscope/EGM.md`, `./actorauthz/EGM.md`, `./actorlookup/EGM.md`.

## Invariants
- A status error from the credential source or already classified by the plugin reaches the caller unchanged; unrecognized errors become `codes.Internal` and are logged without their text (`status.go:toStatus`, `internalErrorAttrs`).
- Transfer clients are keyed by the actor JWT the request carries; a storage refusal (401/403) forces a credential refresh (`credentials.go:refused`, `transfer` doc).
- Uploads are bounded by the configured memory budget (`node.go:NewNodePlugin` `uploadMemory`).

## Side Effects
- Reads and writes snapshot objects (zstd-compressed) and asset objects in the object store.
- Serves the node/asset/control plugin gRPC services and the credential exchange endpoint.
- Stages assets on local disk under the staging dir, bounded by `stagingCapacity` (`asset.go:NewAssetPlugin`).

## Verification
- Run: `go test ./internal/objectstoreplugin/...`
