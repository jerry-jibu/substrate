# podcertcontroller/internal/podidentitysigner

## Overview
Signer `podidentity.podcert.ate.dev/identity`. Issues certificates equivalent to Kubernetes ServiceAccount tokens, embedding a SPIFFE identity, backed by a local CA pool.

## Interface
- `podidentitysigner.go:Name` - const `"podidentity.podcert.ate.dev/identity"`.
- `podidentitysigner.go:Impl` / `podidentitysigner.go:NewImpl` - signer implementation of `signercontroller.SignerImpl`.
- `podidentitysigner.go:DesiredClusterTrustBundles` - PEM bundle of CA trust anchors named `CTBPrefix + "primary-bundle"`.
- `podidentitysigner.go:MakeCert` - issues a cert whose SPIFFE URI is `spiffe://cluster.local/ns/<namespace>/sa/<serviceaccount>`.

## Invariants
- Identity comes from the PCR spec alone; kube-apiserver attests it for kubelet-created requests (`podidentitysigner.go:MakeCert` comment).
- Certificate lifetime is capped at 24h and shortened to the requested `MaxExpirationSeconds` (`podidentitysigner.go:MakeCert`).
- `notBefore` is backdated 2 minutes and `beginRefreshAt` is 30 minutes before `notAfter` (`podidentitysigner.go:MakeCert`).

## Side Effects
- Writes signed certificates through PCR status updates; reads CA pool state via `localca.Pool`.

## Verification
- Run: `go test ./cmd/podcertcontroller/internal/podidentitysigner/...`
