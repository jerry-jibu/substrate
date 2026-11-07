# installdefaults

## Overview

Default namespace, Service, and ServiceAccount names matching the canonical
install layout in `manifests/ate-install/`. Binaries use these as flag
defaults; deployments that diverge pass actual values via flags.

## Interface

- Constants: `SystemNamespace` ("ate-system"), `APIServiceName`, `RouterServiceName`, `IDPServiceName`, `ClientServiceAccount`, `AteletTrustDomain`, `AteletServiceAccount`, `RouterServiceAccount`, `EgressServiceAccount`, `ControllerServiceAccount`, `APIServerServiceAccount`, `TransferServiceAccountName`, `WorkerServiceAccount`, `PodNamespaceEnv` (`installdefaults.go:SystemNamespace`).
- `NamespaceFromPodEnv() string` (`installdefaults.go:NamespaceFromPodEnv`): namespace from the downward-API env var, falling back to SystemNamespace.
- `ActorJWTIssuer(namespace string) string` (`installdefaults.go:ActorJWTIssuer`): default actor JWT issuer URL of the ate-idp-server Service.
- `APIServiceHost(namespace string) string` (`installdefaults.go:APIServiceHost`): in-cluster DNS name of ate-api-server's Service.
- `ClientPrincipal(namespace string) string` (`installdefaults.go:ClientPrincipal`): principal ate-api-server sees for the client ServiceAccount's token.
- `SPIFFEID(namespace, serviceAccount string) string` (`installdefaults.go:SPIFFEID`) and per-component helpers `AteletSPIFFEID`, `RouterSPIFFEID`, `EgressSPIFFEID`, `ControllerSPIFFEID`, `APIServerSPIFFEID` (`installdefaults.go:AteletSPIFFEID`).

## Invariants

- Names and IDs must match the canonical layout in `manifests/ate-install/` (`installdefaults.go` package comment).
- Relying parties bind trust to the exact `ActorJWTIssuer` string (`installdefaults.go:ActorJWTIssuer` doc).
- Peers authenticate by comparing against the exact `SPIFFEID` string (`installdefaults.go:SPIFFEID` doc).

## Side Effects

- `NamespaceFromPodEnv` reads the `POD_NAMESPACE` environment variable (`installdefaults.go:NamespaceFromPodEnv`).

## Verification

- Run: `go test ./internal/installdefaults/...`
