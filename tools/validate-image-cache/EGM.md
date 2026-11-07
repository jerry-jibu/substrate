# validate-image-cache

## Overview

Root-module Go tool that batch-validates OCI images can be pulled, parsed,
and unpacked by `internal/imagecache` (the atelet-side half of the
node-local image cache). It exercises the exact production pull path,
`Store.EnsureImage` (registry pull, parallel per-layer streaming unpack,
whiteout capture, record write), for every ref in a file and reports
per-image results as CSV. It does not mount overlays or run workloads.

## Interface

- `main.go:main` - flags: `--refs-file` (required unless `--evict-all`), `--cache-dir` (required), `--sample`/`--seed` (reproducible sampling), `--out` (CSV, default `validate-results.csv`, written incrementally), `--parallel` (3), `--timeout` (20m per image), `--min-free-gb` (150), `--evict-idle` (10m), `--evict-all`, `--force`, `--platform` (linux/amd64). CSV columns: `ref, digest, layers, seconds, error`; exit non-zero if any image failed (`README.md` in this directory).
- `main.go:looksLikeLiveNode` - any stat result but ENOENT on the actors dir counts as a live node; `main.go:liveNodeIdleFloor` (1m) is the minimum `--evict-idle` there.
- `main.go:errUsage` - marks validation failures that print flag usage.
- Subdirectories: none.

## Invariants

- Disk is bounded by the cache's own eviction engine, not by the tool: below `--min-free-gb` it asks `Store.EvictUnused` to reclaim the shortfall (`main.go` package comment).
- On a node with an actors dir both modes require `--force` and `--evict-idle` is floored at `liveNodeIdleFloor`, because min-age is the only cross-process guard: the engine's locks are per-process, so a run beside a live atelet is unsynchronized with its GC (`main.go` package comment, `README.md` "Live nodes").
- Failures never stop the run; each image is validated independently and the exit code carries the result (`README.md` "Output and rerunning").
- Authentication here is application-default credentials, not atelet's kubelet credential provider, so a corpus that validates locally can still fail on a node (`README.md` "Prerequisites").

## Side Effects

- Pulls images from registries (gcr.io/pkg.dev with application-default credentials, others anonymously), unpacks layers into `--cache-dir` via `internal/imagecache.Store`, evicts from the cache, and writes the results CSV.

## Verification

- Run: `go test ./tools/validate-image-cache/...` (root module)
