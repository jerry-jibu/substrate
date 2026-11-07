# podcertcontroller/internal/podcertificate

## Overview
Client for the Kubernetes PodCertificateRequest (PCR) API. Selects one served API version (v1 preferred, v1beta1 fallback) for both the shared informer cache and status writes, and exposes the signers' beta representation internally.

## Interface
- `client.go:Client` - version-selecting PCR client.
- `client.go:NewClient` - discovers which PCR API version is served and builds the client; fails if neither v1 nor v1beta1 is served.
- `client.go:Informer` - the owned shared informer; the caller must run it once for all controllers sharing this client.
- `client.go:GetCached` - read-only get from the informer cache, converted to the beta representation.
- `client.go:UpdateStatus` - status write through the same API version the informer caches.
- `publickey.go:PublicKey` - extracts the subject public key from a PCR's `StubPKCS10Request`, or legacy `PKIXPublicKey`.

## Invariants
- `client.go:NewClient` prefers v1 and falls back to v1beta1 only when discovery reports v1 absent.
- `client.go:GetCached` results must be treated as read-only, like a standard lister.
- `client.go:toBeta` / `client.go:toV1` deep-copy before converting between representations.

## Side Effects
- `client.go:Informer` watches PodCertificateRequests cluster-wide (all namespaces, 24h resync).
- `client.go:UpdateStatus` issues PCR status updates against the Kubernetes API server.

## Verification
- Run: `go test ./cmd/podcertcontroller/internal/podcertificate/...`
