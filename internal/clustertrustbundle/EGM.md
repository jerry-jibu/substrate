# clustertrustbundle

## Overview
Kubernetes client for ClusterTrustBundle objects that works on clusters serving either the v1 or v1beta1 API, presenting the stable v1 representation to callers.

## Interface
- `client.go:Discover` - prefers v1 and falls back only when discovery reports it absent.
- `client.go:NewClient` - builds a client pinned to one served version, with an optional list-options tweak.
- `client.go:Client` - `Get`, `List`, `Create`, `Update`, `Delete` live operations; `Informer` lazily creates a shared informer; `GetCached` reads from the informer cache.
- `client.go:ToBeta`, `client.go:ToV1` - conversions between the beta and stable representations.
- `client.go:ErrNotServed` - sentinel error when neither version is served.

## Invariants
- All returned objects use the stable v1 representation regardless of the served version (`client.go:Client`).
- `Discover` falls back to v1beta1 only when discovery reports v1 absent (`client.go:Discover`).
- The informer is created lazily; live operations allocate none, and the caller must start it once if cached reads are needed (`client.go:Informer`).

## Side Effects
- Talks to the Kubernetes API server for discovery, live CRUD, and informer watches (`client.go:NewClient`, `client.go:Informer`).

## Verification
- Run: `go test ./internal/clustertrustbundle/...`
