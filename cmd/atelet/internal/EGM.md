# atelet/internal

## Overview

Internal packages of the atelet binary: the on-node path layout and actor
directory lifecycle, node-local content caches and their garbage collection,
snapshot tracking and background upload/detach, sparse file copying, atomic
file writes, image credential resolution, request validation, and trust
bundle resolution.

## Subdirectories & Boundaries

- actordirs: [./actordirs/EGM.md](./actordirs/EGM.md) - empties and reclaims each actor's on-node directory tree across lifecycle transitions.
- apivalidation: [./apivalidation/EGM.md](./apivalidation/EGM.md) - generated declarative validation for the atelet RPC surface plus custom field rules.
- ateletpath: [./ateletpath/EGM.md](./ateletpath/EGM.md) - atelet's on-node path layout and the `ActorDirs` message passed to worker.
- atomicfile: [./atomicfile/EGM.md](./atomicfile/EGM.md) - atomic file writes via temp file plus rename.
- credentialprovider: [./credentialprovider/EGM.md](./credentialprovider/EGM.md) - registry credentials via kubelet image credential provider plugins, as an `authn.Keychain`.
- filecache: [./filecache/EGM.md](./filecache/EGM.md) - node-local disk cache of downloaded artifacts with link-out serving and LRU eviction.
- filecachegc: [./filecachegc/EGM.md](./filecachegc/EGM.md) - per-cache eviction loop with byte budget, flags, and metrics.
- snapshottracker: [./snapshottracker/EGM.md](./snapshottracker/EGM.md) - per-actor snapshot tracking, background upload/detach, and control-plane reporting.
- sparsefile: [./sparsefile/EGM.md](./sparsefile/EGM.md) - hole-preserving file copy and allocated-bytes measurement.
- trustbundle: [./trustbundle/EGM.md](./trustbundle/EGM.md) - resolves allowlisted trust bundle names to PEM certificate bundles.

## Verification

- Run: `go test ./cmd/atelet/internal/...`
