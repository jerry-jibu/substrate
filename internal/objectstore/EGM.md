# objectstore

## Overview

Manages the existence of the objects an external snapshot is made of: listing
them, copying them server-side, and deleting them once nothing owns them. It is
deliberately separate from `pkg/objectstorage`, which reads and writes snapshot
content; the control plane never handles those bytes, so a multi-gigabyte
memory image never transits ate-api (`objectstore.go` package comment).

## Interface

- `Store` (`objectstore.go:Store`): object-storage surface the control plane needs: `List`, `Delete`, `Copy`, all addressing objects by name only.
- `NewS3(client *s3.Client) Store` (`s3.go:NewS3`): Store backed by an S3-compatible object store.
- `NewGCS(client *storage.Client) Store` (`gcs.go:NewGCS`): Store backed by Google Cloud Storage.
- `DeletePrefix(ctx, s Store, uri resources.StoragePrefix) error` (`objectstore.go:DeletePrefix`): removes every object under `uri`.
- `CopyPrefix(ctx, s Store, src, dst resources.StoragePrefix) error` (`objectstore.go:CopyPrefix`): copies every object of the snapshot at `src` to `dst`, keeping names relative to the prefix.
- `BucketPrefix(uri resources.StoragePrefix) (string, string, error)` (`objectstore.go:BucketPrefix`): splits a storage prefix into bucket and object-name prefix.
- `ErrEmptySource` (`objectstore.go:ErrEmptySource`): sentinel for CopyPrefix finding no objects to copy.
- Subdirectory:
  - [objectstoretest](./objectstoretest/EGM.md) - in-memory `Store` fake for tests.

## Invariants

- Deleting an already-deleted object succeeds, so a retry over a partly-collected snapshot finishes cleanly (`objectstore.go:Store.Delete` doc, `s3.go:Delete`, `gcs.go:Delete`).
- Copy destinations are deterministic and overwritten, so a retry of a partly-completed copy overwrites instead of duplicating (`objectstore.go:CopyPrefix`, `objectstore.go:Store.Copy` doc).
- An empty copy source fails with `ErrEmptySource`; copying nothing would leave an unrestorable snapshot (`objectstore.go:CopyPrefix`).
- A missing bucket is treated as already deleted during cleanup, but only for provider errors that identify it; other failures still block completion (`objectstore.go:isMissingBucket`).
- Per-object operations run at most 8 at a time per prefix (`objectstore.go:prefixConcurrency`).
- `BucketPrefix` keeps the trailing separator on the prefix so it cannot match a sibling whose name it happens to prefix (`objectstore.go:BucketPrefix`).
- S3 objects at or above 5 GiB copy via multipart with 1 GiB parts and at most 4 parts in flight; a failed or uncompletable multipart copy aborts so no abandoned upload keeps billing (`s3.go:copyObjectMaxSize`, `s3.go:copyMultipart`, `s3.go:abortMultipart`).
- GCS handles retry transient errors (408/429/5xx) always, up to 7 attempts with 250ms-5s backoff (`gcs.go:gcsBackoff`, `gcs.go:bucket`).

## Side Effects

- Performs provider API calls: list, delete, and server-side copy against S3-compatible stores and GCS (`s3.go`, `gcs.go`); bytes never transit this process.
- Runs concurrent object and part operations with errgroup (`objectstore.go:DeletePrefix`, `s3.go:copyParts`).
- Emits slog warnings for missing buckets during deletion.

## Verification

- Run: `go test ./internal/objectstore/...`
