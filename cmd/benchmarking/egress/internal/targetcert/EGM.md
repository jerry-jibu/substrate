# targetcert

## Overview

Issues the name-constrained CA and the HTTPS certificate the egress test
target serves, for the driver to deploy and the actor to verify against.
The CA vouches for nothing but names directly under the endpoint domain:
the egress gateway is shared, so the CA is name-constrained to the domain
and its private key is discarded after signing.

## Interface

- `targetcert.go:Bundle` - CA, certificate, and key, all PEM-encoded (`CA`, `Cert`, `Key`).
- `targetcert.go:Generate` - issues a new CA and a wildcard leaf for `*.<domain>`, valid from `now`.
- `targetcert.go:Bundle.Write` - writes `ca.crt`, `tls.crt` (0644) and `tls.key` (0600) into `dir`, the key names the deploy script's TLS Secret expects.
- Subdirectories: none.

## Invariants

- The CA template sets `MaxPathLenZero` and critical `PermittedDNSDomains` limited to `domain`, so a verifier that cannot enforce the constraint rejects the CA instead of trusting it everywhere (`targetcert.go:generateWithCA`).
- The CA private key is thrown away once the leaf is signed; there is no rotation (`targetcert.go:Generate`, `targetcert.go:validity` of 5 years).
- `NotBefore` is backdated by `targetcert.go:clockSkew` (1 hour) for node clock skew.
- The leaf is an ECDSA P-256 server-auth certificate with a random 128-bit serial (`targetcert.go:generateWithCA`, `targetcert.go:serialNumber`).

## Side Effects

- `Bundle.Write` creates `dir` (0700) and writes three PEM files to disk (`targetcert.go:Bundle.Write`).

## Verification

- Run: `go test ./cmd/benchmarking/egress/internal/targetcert/...`
