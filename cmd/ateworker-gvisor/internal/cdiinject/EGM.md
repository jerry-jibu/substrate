# cdiinject

## Overview
Applies CDI container edits to an actor's OCI bundle for gVisor ("Package cdiinject", `inject.go`). The gVisor shape writes the bundle on disk and does not generalize to other runtimes.

## Interface
- `inject.go:IntoBundle(ctx context.Context, bundleDir string, spec *cdi.Spec, opts Options) error` rewrites the bundle's OCI spec in place with the CDI device edits; `inject.go:Options` carries the knobs.
- `soname.go:StageSonameSymlinks(ctx context.Context, rootfs string, mounts []specs.Mount, allowedHostDirs []string) error` stages shared-library soname symlinks the sandbox needs.

## Invariants
- Device nodes must live under the configured host directory (`inject.go:devRoot` doc); mounts and devices outside the allowed host dirs are rejected (`inject.go:validateMount`, `inject.go:resolveAllowedHostPath`).
- Injection is idempotent per device: an already-present device node is detected by `inject.go:hasInjectedDevice` and not duplicated.

## Side Effects
- Writes the OCI bundle files under `bundleDir`.
- Reads ELF sonames from the rootfs (`soname.go:elfSoname`) and creates symlinks in it.

## Verification
- Run: `go test ./cmd/ateworker-gvisor/internal/cdiinject/...`
