# objectstorage

## Overview
Snapshot object-store I/O shared by atelet and the objectstore plugins. Wraps GCS and S3 behind one `ObjectStorage` interface and adds the transfer path used for actor snapshots: zstd compression (plain and sparse-extent formats), parallel ranged downloads, parallel part uploads with server-side compose on GCS, and a shared memory budget across concurrent uploads. `NewFromEnv` picks the backend from `ATE_STORAGE_BACKEND`.

## Interface
- `ObjectStorage` with `GetObject(ctx, bucket, object)` and `PutObject(ctx, bucket, object, reader)` (`objects.go:ObjectStorage`)
- `NewGCSClient(ctx, opts...)`, `NewS3Client(client)`, `NewFromEnv(ctx)`, `UsesS3()`, `NewS3ClientFromEnv(ctx)` (`gcs.go:NewGCSClient`, `s3.go:NewS3Client`, `env.go:NewFromEnv`, `env.go:UsesS3`, `env.go:NewS3ClientFromEnv`)
- `Uploader` with `NewUploader(client, capacity)`, `WithClient(client)`, `SendFile(ctx, url, file)`, `Send(ctx, url, content)`; `DefaultUploadMemory` budget constant (`upload.go:Uploader`, `upload.go:NewUploader`, `upload.go:WithClient`, `upload.go:SendFile`, `upload.go:Send`, `upload.go:DefaultUploadMemory`)
- Fetch/put helpers: `FetchFromGCS`, `Open`, `SendLocalFileToGCSWithZstd`, `FetchLocalFileFromGCSWithZstd`, `FetchFileFromGCSWithZstd` (`objects.go:FetchFromGCS`, `objects.go:Open`, `objects.go:SendLocalFileToGCSWithZstd`, `objects.go:FetchLocalFileFromGCSWithZstd`, `objects.go:FetchFileFromGCSWithZstd`)
- `SizedReader`/`ObjectSize(r)` for size-reporting readers (`objects.go:SizedReader`, `objects.go:ObjectSize`)
- Errors: `ErrObjectNotFound`, `ServiceError`, `DescribeServiceError(err)` (`objects.go:ErrObjectNotFound`, `objects.go:ServiceError`, `objects.go:DescribeServiceError`)
- Sparse format primitives: `writeSparseZstd`/`writeSparseExtents`, `readSparseZstd`, `sparseMagic`, `sparseVersion`, `sparseExtents`, `planSparseParts`, `writeSparsePart`, `sparsePartCount` (`sparsezstd.go:writeSparseZstd`, `sparsezstd.go:readSparseZstd`, `sparsezstd.go:sparseMagic`, `sparsezstd.go:sparseVersion`, `sparseparts.go:sparseExtents`, `sparseparts.go:planSparseParts`, `sparseparts.go:writeSparsePart`, `sparseparts.go:sparsePartCount`)
- Parallel zstd: `parZstd` writer with `newParZstd`/`newParZstdChunk`, inline mode at one worker, `parZstdChunk`/`parZstdMinChunk`/`parZstdQueue` constants (`parzstd.go:parZstd`, `parzstd.go:newParZstd`, `parzstd.go:newParZstdChunk`)

## Invariants
- `GetObject` maps a missing object or bucket to an error wrapping `ErrObjectNotFound`, not a transient failure (`gcsranged.go:GetObject`, `s3ranged.go:GetObject` with `objectAbsent`).
- GCS writes are retried with `storage.RetryAlways` and no attempt cap, so every caller must bound its context; resumable uploads also give up per the SDK's 32s `ChunkRetryDeadline` (`gcs.go:setRetry`).
- The sparse-extent format is `magic[8] | version:u32 | zstd(totalSize | (off,len,data)* | -1)`; `sparseMagic` is deliberately not a zstd frame magic, and `sparseVersion` must be bumped on incompatible layout change (`sparsezstd.go:writeSparseZstd`, `sparsezstd.go:sparseMagic`, `sparsezstd.go:sparseVersion`).
- `readSparseZstd` rejects a stream whose declared version differs and validates each extent against the declared total size before writing (`sparsezstd.go:readSparseZstd`).
- Composed multipart uploads are byte-identical to a single `PutObject` because parts are consecutive ranges of the same stream (`gcscompose.go:putComposite`, `gcssparse.go:PutSparseFile`).
- `NewUploader` rejects a capacity below the backend's buffers plus one compression worker (`upload.go:NewUploader`).
- Reservation accounting: uploads reserve memory for compression workers plus backend buffers; peak S3 buffering is `uploadPartSize * uploadConcurrency` (128 MiB) (`upload.go:uploadBufferMemory`, `s3.go:uploadPartSize`, `s3.go:uploadConcurrency`).
- `parZstd.Close` reports the first worker or destination error, and later writes return `os.ErrClosed` (`parzstd.go:Close`, `parzstd.go:markClosed`).
- `DescribeServiceError` never surfaces a URI, header, or credential; backends only describe what they judge safe to log (`objects.go:ServiceError`).

## Side Effects
- Network I/O to GCS or S3 for every get/put, including ranged reads and part uploads (`gcsranged.go:fetchRange`, `s3ranged.go:fetchRange`, `gcscompose.go:putComposite`).
- Composite uploads write scratch objects (`<object>.part-<runID>-<NNNN>` and `.compose-...`) and delete them best-effort within `cleanupTimeout` (30s); failures leave scratch behind and only log (`gcscompose.go:removeScratch`, `gcscompose.go:cleanupTimeout`).
- `sendBufferedZstd` stages compressed content in a temp file removed after upload (`objects.go:sendBufferedZstd`).
- Local file mutation: `decodeContent`/`copyZstdSparse` truncate the destination file and write holes for zero blocks; `readSparseZstd` truncates to the declared size (`objects.go:copyZstdSparse`, `sparsezstd.go:readSparseZstd`).
- Created files are chmod 0600 by the fetch helpers (`objects.go:FetchLocalFileFromGCSWithZstd`).
- Goroutines: ranged readers, part uploads, and `parZstd` workers run on background pools bounded by `downloadConcurrency`, `uploadConcurrency`, and the worker count (`rangedget.go:newRangedReader`, `gcscompose.go:putComposite`, `parzstd.go:newParZstdChunk`).
- Reads `ATE_STORAGE_BACKEND` and standard AWS env vars; `AWS_S3_USE_PATH_STYLE=true` selects path-style addressing (`env.go:UsesS3`, `env.go:NewS3ClientFromEnv`).

## Verification
- Run: `go test ./pkg/objectstorage/...` from the repo root.
