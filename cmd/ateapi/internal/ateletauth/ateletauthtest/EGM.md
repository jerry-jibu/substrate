# ateapi/internal/ateletauth/ateletauthtest

## Overview

Builds the peer contexts that the ateapi services served only to atelet
authenticate against (`ateletauthtest.go` package doc).

## Interface

- `Cert` (`ateletauthtest.go`) - self-signed certificate carrying a SPIFFE path and optional PodIdentity; either may be omitted to produce a certificate authentication must reject.
- `PodIdentityOn` / `PodIdentityIn` (`ateletauthtest.go`) - a well-formed atelet PodIdentity pinned to a node, in the default or a named namespace.
- `CertOn` / `CertIn` (`ateletauthtest.go`) - the certificate of an atelet on a node, in the default or a named namespace.
- `ContextWith` (`ateletauthtest.go`) - inject a certificate as the transport-authenticated peer; a nil cert yields a context with no peer information, which is what an unauthenticated call looks like.

## Invariants

- Certificates carry the atelet trust domain from `installdefaults.AteletTrustDomain` and the atelet ServiceAccount from `installdefaults.AteletServiceAccount` (`ateletauthtest.go` `Cert`, `PodIdentityIn`).

## Side Effects

- ["None"]

## Verification

- Run: `go test ./cmd/ateapi/internal/ateletauth/ateletauthtest/...`
