# pemutil

## Overview

Sanitizes PEM certificate bundles for projection into actors, the way
kubelet does for clusterTrustBundle projected volumes.

## Interface

- `pemutil.go:SanitizeCertificateBundle` - re-encodes a PEM bundle keeping only CERTIFICATE blocks, stripping block headers and deduplicating by DER bytes; errors when the input holds no CERTIFICATE blocks.
- Subdirectories: none.

## Invariants

- An empty trust bundle is an error, never silently passed through (`pemutil.go:SanitizeCertificateBundle`).
- Output order is meaningless: certificates are sorted first, then deterministically shuffled by `seed`, so identical inputs and seeds yield identical outputs and order dependence is detectable (`pemutil.go:SanitizeCertificateBundle`).

## Side Effects

- ["None"] (pure byte transformation)

## Verification

- Run: `go test ./internal/pemutil/...`
