# fakeworkertest

## Overview

Test helper: issues pod-identity-shaped certificates for tests of the fake
data plane's capacity relay. A throwaway CA and leaf certificates that
carry a SPIFFE ID as their only SAN, like the pod-identity signer's.

## Interface

- `fakeworkertest.go:CA` - throwaway signer backed by `localca.ConcretePool` with files in the test's temp dir; `CA.TrustBundle` is the PEM bundle path.
- `fakeworkertest.go:NewCA` - creates the CA (ECDSA P-256, 1 hour validity) and writes `trust-bundle.pem`.
- `fakeworkertest.go:CA.Issue` - writes a credential bundle (certificate, then PKCS #8 key) for a SPIFFE ID and returns its path.
- `fakeworkertest.go:Rotate` - replaces a trust bundle by renaming a new file over the old, as kubelet rotates a projected bundle.
- Subdirectories: none.

## Invariants

- Leaves carry the SPIFFE ID as their only SAN and are valid for 1 hour (`fakeworkertest.go:CA.Issue`).
- Files are written 0600 (`fakeworkertest.go:writePEM`).
- Testing-only package; requires `*testing.T`.

## Side Effects

- Writes PEM files and performs rename-based rotation inside the test temp dir (`fakeworkertest.go:NewCA`, `fakeworkertest.go:CA.Issue`, `fakeworkertest.go:Rotate`).

## Verification

- Run: `go test ./cmd/benchmarking/isolate/internal/fakeworker/...`
