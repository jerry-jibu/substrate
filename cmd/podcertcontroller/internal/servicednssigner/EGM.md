# podcertcontroller/internal/servicednssigner

## Overview
Signer `servicedns.podcert.ate.dev/identity`. Issues certificates for Kubernetes service DNS names covering the requesting pod, backed by a local CA pool, and publishes the CA trust anchors as a ClusterTrustBundle.

## Interface
- `servicednssigner.go:Name` - const `"servicedns.podcert.ate.dev/identity"`.
- `servicednssigner.go:Impl` / `servicednssigner.go:NewImpl` - signer implementation; satisfies `signercontroller.SignerImpl`.
- `servicednssigner.go:SignerName` - returns `Name`.
- `servicednssigner.go:DesiredClusterTrustBundles` - PEM bundle of the CA pool trust anchors named `CTBPrefix + "primary-bundle"`.
- `servicednssigner.go:MakeCert` - lists Services in the PCR namespace and issues a cert for the DNS names of services whose selector covers the requesting pod.

## Invariants
- `servicednssigner.go:MakeCert` only considers Service types ClusterIP, NodePort, and LoadBalancer with a non-empty selector.
- `signercontroller.SignerImpl` compliance is asserted at compile time (`var _ signercontroller.SignerImpl = (*Impl)(nil)`).

## Side Effects
- Reads Services from the Kubernetes API and writes signed certificates back through PCR status.
- Reads CA pool state (refreshed from the pool file) via `localca.Pool`.

## Verification
- Run: `go test ./cmd/podcertcontroller/internal/servicednssigner/...`
