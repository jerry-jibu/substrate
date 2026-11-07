# podidentity

## Overview

Requests certificates from Substrate's podidentity signer for processes
that do not run in a pod, via Kubernetes `PodCertificateRequest` objects.

## Interface

- `podidentity.go:SignerName` - `podidentity.podcert.ate.dev/identity`, the signer issuing the SPIFFE certificates the ingress router verifies.
- `podidentity.go:RequestCertificate` - generates an ECDSA P-256 key locally, creates a `PodCertificateRequest` (stub CSR, `MaxExpirationSeconds` 24h), polls for Issued/Denied/Failed conditions (500ms interval, caller timeout), deletes the request, and returns a `tls.Certificate` after checking the issued leaf carries the requested public key.
- `podidentity.go:pcrClient` (`podCertificateRequests`) - creates/reads/deletes PodCertificateRequests through certificates.k8s.io v1 or v1beta1, preferring v1.
- Subdirectories: none.

## Invariants

- The private key never leaves this process; the CSR is a stub carrying only the public key (`podidentity.go:RequestCertificate`).
- The issued certificate must carry the requested public key or the call fails (`podidentity.go:RequestCertificate` public-key check).
- The pod, ServiceAccount, and node named in the request do not exist; the signer takes the identity from the request, and `requester` names the maker in the request's generateName (`podidentity.go:RequestCertificate` doc).

## Side Effects

- Creates and deletes PodCertificateRequests in the Kubernetes API; a leftover request is harmless (garbage collected) (`podidentity.go:RequestCertificate` defer, `podidentity.go:pcrClient.create`/`delete`).
- Performs API discovery of the served certificates.k8s.io versions (`podidentity.go:podCertificateRequests`).

## Verification

- Run: `go test ./internal/podidentity/...`
