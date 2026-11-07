# trustbundle

## Overview

Resolves the trust bundle names that actors may project into system-info
volumes to PEM certificate bundles, from an allowlist of well-known names.

## Interface

- `trustbundle.go:EgressName` - well-known name (`egress-mitm.ate.dev`) of the egress gateway CA bundle.
- `trustbundle.go:EgressCTB` - the `ClusterTrustBundle` object read for `EgressName`.
- `trustbundle.go:SystemRootsName` - well-known name (`system-roots.ate.dev`) for a selection of WebPKI roots.
- `trustbundle.go:NewSource` - creates a `Source` from a ClusterTrustBundle getter and the system roots PEM.
- `trustbundle.go:(*Source).Combined` - returns the sanitized, deduplicated union of the named bundles' anchors.

## Invariants

- Only the two allowlisted names resolve; anything else is an unknown trust bundle error (`trustbundle.go:(*Source).raw`).
- `Combined` requires at least one name (`trustbundle.go:(*Source).Combined`).
- Bundle contents are sanitized and anchors deduplicated across bundles via `internal/pemutil.SanitizeCertificateBundle` with a per-Source random shuffle seed (`trustbundle.go:(*Source).Combined`).

## Side Effects

- Calls the injected `getCTB` function to read `ClusterTrustBundle` objects (a Kubernetes API read in production) (`trustbundle.go:(*Source).raw`).

## Verification

- Run: `go test ./cmd/atelet/internal/trustbundle/...`
