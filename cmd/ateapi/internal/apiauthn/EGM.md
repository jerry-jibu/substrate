# ateapi/internal/apiauthn

## Overview

Authenticates clients of the ateapi gRPC server. The interceptor takes identity
from the transport-layer mTLS credentials when the client presented a
certificate, and otherwise requires an authorization header
`Bearer <JWT Token>`. Requests with no credentials are rejected
(`apiauthn.go` package doc).

## Interface

- `UnaryServerInterceptor`, `StreamServerInterceptor` (`apiauthn.go`) - gRPC interceptors enforcing a `ServerConfig`.
- `ServerConfig`, `JWTProvider` (`apiauthn.go`) - the trusted JWT providers that verify bearer tokens for clients without a certificate identity (e.g. kubectl-ate).
- `ValidateServerConfig` (`apiauthn.go`) - at least one provider is required, each needs a name, issuer, and verifier, and issuers must be unique.
- `chainedServerAuthenticator.authenticate` / `mtlsPeerIdentity` (`apiauthn.go`) - unexported chain: mTLS peer first, then JWT; identity is injected with `principal.InjectContext`.

## Invariants

- Every RPC must carry either a verified client certificate or a bearer token a configured provider verifies; anything else is rejected (`apiauthn.go` package doc, `authenticate`).
- mTLS identity is the first URI SAN (a SPIFFE ID) of the transport-authenticated peer certificate (`apiauthn.go` `mtlsPeerIdentity`).

## Side Effects

- ["None"] beyond rejecting requests and annotating the context.

## Verification

- Run: `go test ./cmd/ateapi/internal/apiauthn/...`
