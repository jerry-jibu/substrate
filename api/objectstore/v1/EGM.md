# objectstore/v1

## Overview

The `substrate.objectstore.v1` proto package defines the plugin APIs an
object storage platform implements for Agent Substrate: node-side snapshot
transfer (`NodeProvider`), control-plane snapshot management
(`ControlProvider`), and content-addressed sandbox asset serving
(`AssetProvider`). `objectstore.proto` is the source of truth; the two
`*.pb.go` files are generated. Go package:
`go.ate.dev/substrate/api/objectstore/v1` (package `objectstore`). The
in-tree plugin is `cmd/objectstore-plugin`.

## Interface

- `service NodeProvider` (`objectstore.proto:36`): `FetchSnapshot` downloads
  named snapshot files into `write_path`; `UploadSnapshot` uploads named
  files from `local_path`, succeeding only once every file is durably stored.
- `service ControlProvider` (`objectstore.proto:101`): `CleanupSnapshot`
  deletes every file under a snapshot or owner prefix; `CopySnapshot` copies
  every file from `src_uri` to `dst_uri`, across buckets.
- `service AssetProvider` (`objectstore.proto:146`): `FetchAsset` writes the
  object at `asset_uri` to `write_path` only if its content hashes to the
  given `sha256`, bounded by `max_bytes`.
- Request/response messages: `FetchSnapshotRequest`, `UploadSnapshotRequest`,
  `CleanupSnapshotRequest`, `CopySnapshotRequest`, `FetchAssetRequest` and
  their empty responses (`objectstore.proto:52`-`:180`).
- Generated Go stubs: `NodeProviderClient/Server`, `ControlProviderClient/Server`,
  `AssetProviderClient/Server` and their `Register*Server` functions
  (`objectstore_grpc.pb.go:59`, `:120`, `:237`, `:282`, `:404`, `:450`,
  `:158`, `:316`, `:482`), plus message structs in `objectstore.pb.go`.

## Invariants

- Every RPC is idempotent: a retry overwrites whatever an earlier attempt
  stored or wrote (`objectstore.proto:34`-`:35`).
- The caller owns commit ordering: data files upload first, the manifest
  last, so a reader that finds the manifest can rely on every file it lists
  (`objectstore.proto:28`-`:32`).
- Substrate stores no checksums; integrity verification is the plugin's
  (`objectstore.proto:26`-`:27`).
- `FetchSnapshot` returns NOT_FOUND if any requested file is absent;
  per-actor-authorizing plugins return PERMISSION_DENIED when `actor_jwt`
  does not grant access (`objectstore.proto:38`-`:48`).
- `actor_jwt` is a worker-for-actor token with audience
  `objectstore-plugin.ate.dev`; plugins without per-actor authorization
  ignore it (`objectstore.proto:56`-`:63`).
- `FetchAsset` writes nothing unless the sha256 matches, refuses any URI
  under a snapshot or tag location with PERMISSION_DENIED before reading, and
  returns FAILED_PRECONDITION when the object exceeds `max_bytes` or the hash
  mismatches (`objectstore.proto:147`-`:155`).
- A bare bucket passed to `CleanupSnapshot` is rejected with
  INVALID_ARGUMENT (`objectstore.proto:116`-`:117`).
- `asset_uri` and `actor_jwt` are marked `debug_redact` (`objectstore.proto:63`,
  `:163`).

## Side Effects

- The `Register*Server` functions attach handlers to a `grpc.ServiceRegistrar`
  (`objectstore_grpc.pb.go:158`, `:316`, `:482`); the package is otherwise
  pure type definitions.

## Verification

- Run: `go test ./api/...`
