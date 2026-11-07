# imagecache

## Overview

The node-local OCI image cache: a content-addressed pool of unpacked image
layers shared by every actor on the node, plus the per-bundle overlay spec that
tells the worker runtimes how to compose an actor rootfs from cached layers.
Work is split along the atelet/worker privilege boundary: atelet (root) pulls
and unpacks layers and writes bundle specs; workers finalize layers (whiteout
materialization needs CAP_SYS_ADMIN) and mount the overlay rootfs
(`imagecache.go` package comment). `bundle_linux.go` is Linux-only.

## Interface

- `Store` / `New(root string, opts ...Option) (*Store, error)` (`imagecache.go:New`): opens the on-disk pool, verifies the layout version, sweeps crash debris, and runs startup orphan recovery.
- Options: `WithKeychain`, `WithLocalhostRegistryReplacement`, `WithPlatform`, `WithActorsDir`, `WithMinAge`, `WithPullTimeout`, `WithMeter` (`imagecache.go:Option`); `CandidateKeychain` (`imagecache.go:CandidateKeychain`) supplies ordered pull credentials.
- `(*Store).EnsureImage(ctx, ref string) (*Image, error)` (`imagecache.go:EnsureImage`): makes an image available and returns its config and ordered layer dirs.
- `Image` (`imagecache.go:Image`) with `(*Image).ReadFile(name string) ([]byte, error)` (`imagefile.go:ReadFile`): reads a file as the composed image would present it, without mounting.
- `(*Store).CacheSize() (int64, error)` (`sizes.go:CacheSize`): sum of recorded layer sizes in the pool.
- `(*Store).InUse() (RootSet, error)` (`gc.go:InUse`): images and layers referenced by bundle overlay specs under the actors dir.
- `(*Store).EvictUnused(ctx, targetBytes int64, dryRun bool) (EvictStats, error)` (`gc.go:EvictUnused`): LRU eviction of unprotected images and unreferenced layers.
- `(*Store).RecoverOrphans(ctx) (EvictStats, error)` (`gc.go:RecoverOrphans`): startup-only reclamation of unreferenced layer dirs.
- `ErrIncompleteEnumeration` (`gc.go:ErrIncompleteEnumeration`): sentinel for a pass gated on partial enumeration.
- `OverlaySpecFileName`, `OverlaySpec`, `ImageVolumeOverlay` (`spec.go:OverlaySpec`): the atelet-to-worker rootfs contract.
- `WriteSpec(bundlePath, spec)` (`spec.go:WriteSpec`) / `ReadSpec(bundlePath)` (`spec.go:ReadSpec`) / `ImageVolumeMountPath(bundlePath, volumeName)` (`spec.go:ImageVolumeMountPath`).
- `FinalizeLayer(layerDir string) error` (`bundle_linux.go:FinalizeLayer`): materializes recorded whiteouts and opaque markers (worker side, idempotent).
- `SetupBundleRootfs(bundlePath string) error` (`bundle_linux.go:SetupBundleRootfs`): composes and mounts the bundle's overlay rootfs (worker side).
- `UnmountAllUnder(dir string) error` (`bundle_linux.go:UnmountAllUnder`): lazily detaches every mount at or below dir.
- `RemoveAllWritable(path string) error` (`removeall.go:RemoveAllWritable`): removal of image-owned trees with read-only or foreign-owned directories.

## Invariants

- Layers land in the pool via unpack-into-tempdir plus atomic rename, so an existing layer dir is always complete; startup recovery only sweeps `.tmp-`/`.rm-` dirs and orphaned record temp files (`imagecache.go:sweepTempDirs`).
- `New` refuses a pool whose layout version is not "3" rather than mixing layouts (`imagecache.go:layoutVersion`).
- The image record is written before any layer is unpacked, so every layer is referenced before it exists and unexplained layers can only be crash debris (`gc.go` package comment, `imagecache.go:pullWith`).
- Layer content is verified against the config's claimed diffID; a mismatch fails the pull and is never retried (`verify.go:readVerifiedLayer`, `verify.go:errDiffIDMismatch`).
- Digest refs hit the cache with no network I/O; tag refs cost one HEAD to pin an immutable digest as the cache key (`imagecache.go:EnsureImage`).
- Pulls run detached from callers' contexts, bounded only by the store's pull timeout, so a waiter sees only real pull failures (`imagecache.go:EnsureImage`).
- Eviction vetoes, in order: bundle-spec root set, min-age (default 2m), and per-victim re-checks against current disk state; passes are serialized (`gc.go:EvictUnused`).
- Incomplete record or bundle-spec enumeration gates the whole pass toward retention (`gc.go:ErrIncompleteEnumeration`, `gc.go:listEviction`).
- Layer deletion is two-phase: rename to `.rm-` under the pin lock, then RemoveAll outside all locks; a layer pinned by an in-flight pull or with a fresh mtime is never retired (`retire.go:retireLayer`).
- Whiteout entries are recorded in per-layer `whiteouts.json` rather than materialized at unpack, because opaque dirs need `trusted.*` xattrs (CAP_SYS_ADMIN); `FinalizeLayer` materializes them once per layer, idempotently (`bundle_linux.go:FinalizeLayer`, `unpack.go:whiteoutSet`).
- Unpack preserves numeric owners and modes from tar headers; a failed chown or chmod fails the unpack when root (`unpack.go:unpackLayer`); device nodes are skipped so an image cannot grant device access (`unpack.go:unpackLayer`).
- Spec writes and record writes are atomic temp-file-plus-rename so concurrent GC readers never see a partial file (`spec.go:WriteSpec`, `imagecache.go:writeRecord`).
- Overlay lowerdirs are deduplicated (repeated identical layers) and passed one fsconfig(2) call each, avoiding mount(2)'s page-size option cap; overlayfs "lowerdir+" needs Linux 6.7+ or 6.6.23+ (`spec.go:overlayLowerDirs`, `bundle_linux.go:mountOverlay`).
- `Image.ReadFile` is capped at 1 MiB and resolves symlinks within their own layer (`imagefile.go:maxImageFileSize`).
- The `ate.imagecache.requests` metric bounds `error.type` to a fixed status set; a store without a meter records nothing (`metrics.go:reportedStatuses`).

## Side Effects

- Pulls image manifests, configs, and layer blobs from registries with per-registry retry backoffs (fast for dedicated registries, long full-jitter waits for communal ones) (`imagecache.go:retryBackoffFor`).
- Writes the on-disk pool: layer dirs with `fs/`, `whiteouts.json`, `size`, `finalized`; manifest records; the `version` marker (`imagecache.go` package comment).
- Writes `rootfs-overlay.json` into actor bundles (`spec.go:WriteSpec`).
- Worker-side filesystem mutation: mounts overlay rootfs and read-only image volumes in the caller's mount namespace, creates 0:0 char-device whiteouts and `trusted.overlay.opaque` xattrs, repairs implicit-dir metadata into the bundle's private upper (`bundle_linux.go:SetupBundleRootfs`, `implicitdirs.go:applyDirFixups`).
- Reads `/proc/self/mountinfo` for teardown (`bundle_linux.go:mountPointsUnder`).
- Emits the `ate.imagecache.requests` OTel counter and slog logs for hits, misses, evictions, and vetoes (`metrics.go:recordRequest`).

## Verification

- Run: `go test ./internal/imagecache/...`
