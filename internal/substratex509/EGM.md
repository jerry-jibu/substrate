# substratex509

## Overview

Creates and parses x509 certificates embedding Substrate-specific
extensions that communicate a workload's identity. Modeled on upstream
Kubernetes `component-helpers/kubernetesx509`, but extension values are
JSON, not ASN.1.

## Interface

- `substratex509.go:GoogleSubstratePEN` - the PEN arc `1.3.6.1.4.1.11129.2.12` naming Substrate extensions; `substratex509.go:oidPodIdentity` (sub-oid 1) and `substratex509.go:oidActorIdentity` (sub-oid 2).
- `substratex509.go:PodIdentity`, `AddPodIdentityToCertificate`, `PodIdentityFromCertificate` - the Kubernetes pod identity extension (namespace, service account, pod, node, all with UIDs).
- `substratex509.go:ActorIdentity`, `AddActorIdentityToCertificate`, `ActorIdentityFromCertificate` - the actor identity extension (`Atespace`, `ActorName`, `ActorUid`).
- `validatePodIdentity`/`validateActorIdentity` - reject empty fields on both add and parse.

## Invariants

- At most one of each extension may appear on a certificate; more than one is a parse error (`substratex509.go:PodIdentityFromCertificate`, `substratex509.go:ActorIdentityFromCertificate`).
- A certificate without the extension parses to `(nil, nil)` rather than an error.
- Identities are validated on both sides of the wire: adding and parsing both reject empty fields (`validatePodIdentity`, `validateActorIdentity`).

## Side Effects

- ["None"] (mutates only the caller's certificate template via `ExtraExtensions`)

## Verification

- Run: `go test ./internal/substratex509/...`
