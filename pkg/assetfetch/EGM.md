# assetfetch

## Overview
The checks every object-store plugin applies when it serves `AssetProvider.FetchAsset`, whatever its storage backend. A plugin parses its own URI scheme into an object name and an opener and leaves the rest to this package: `CheckObjectName` refuses snapshot and tag locations before anything is read, and `Stager.Fetch` downloads into a plugin-private staging directory within a capacity, enforces `max_bytes`, verifies the sha256, and only then swaps the caller's file. Errors are gRPC statuses with the codes `objectstore/v1` documents for FetchAsset.

## Interface
- `ParseURI(uri) (*url.URL, error)` bounds the URI at `MaxURIBytes` (2048) and rejects user info, query, fragment, and opaque URIs (`assetfetch.go:ParseURI`, `assetfetch.go:MaxURIBytes`)
- `CheckObjectName(uri, base, object) error` enforces canonical object names and refuses snapshot/tag locations (`assetfetch.go:CheckObjectName`)
- `Stager` with `NewStager(root, stagingDir, capacity)` and `Fetch(ctx, req, uri, open, toStatus)`; `OpenFunc` is the backend opener hook (`assetfetch.go:Stager`, `assetfetch.go:NewStager`, `assetfetch.go:Stager.Fetch`, `assetfetch.go:OpenFunc`)
- Platform swap: `exchange` uses `renameat2(RENAME_EXCHANGE)` on Linux and a check-then-rename fallback elsewhere (`exchange_linux.go:exchange`, `exchange_other.go:exchange`)

## Invariants
- Snapshot files, manifests, and tags are never assets: `CheckObjectName` returns PermissionDenied for any `<root>/atespaces/<atespace>/actors/<uid>/snapshots/<name>` or `<root>/atespaces/<atespace>/tags/<uid>` prefix, and the check runs before anything is read (`assetfetch.go:CheckObjectName`, `assetfetch.go:snapshotPrefixLen`).
- The file at `write_path` keeps its old content on any error: the asset is written to a new file and swapped into place, so a descriptor opened before `Fetch` returns reads the old file (`assetfetch.go:Stager.Fetch` doc comment).
- `stagingDir` must be absolute and outside `root`, so the caller can never read an unverified download; `NewStager` removes previous runs' downloads from it (`assetfetch.go:NewStager`).
- `write_path` must be absolute, below `root`, and an existing regular file, not a symlink, directory, or FIFO (`assetfetch.go:writePath`, `assetfetch.go:checkWriteFile`).
- `Fetch` validates sha256 (64 lower-case hex) and positive `max_bytes` before downloading, so a bad request costs no transfer; downloads never hold more than the staging capacity (`assetfetch.go:Stager.Fetch`, `assetfetch.go:reservation`).
- If a download waits for staging space and reopens the object, a changed size fails with Unavailable (`assetfetch.go:Stager.Fetch`).
- On Linux the swap is `RENAME_EXCHANGE`, which fails with `ErrNotExist` if either entry is missing and never creates the target; the non-Linux fallback may recreate a concurrently removed target, and plugins run on Linux (`exchange_linux.go:exchange`, `exchange_other.go:exchange`).
- `ParseURI` errors never quote the URI, since user info or a query may carry credentials (`assetfetch.go:ParseURI` doc comment).

## Side Effects
- Writes and removes files in `stagingDir` (temp downloads named `asset-*`) and creates, swaps, and removes temp files next to the write path (`assetfetch.go:stage`, `assetfetch.go:replace`, `assetfetch.go:tempName`).
- Replaces the caller's file at `write_path` with the verified asset, preserving its mode (`assetfetch.go:replace`).
- Reads the object through the caller's `OpenFunc`, possibly twice when waiting for staging space (`assetfetch.go:Stager.Fetch`).
- Blocks on a `semaphore.Weighted` for staging capacity (`assetfetch.go:Stager` field `reserved`).

## Verification
- Run: `go test ./pkg/assetfetch/...` from the repo root.
