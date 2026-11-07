# oidcjwt

## Overview

Verifies JWTs using OIDC discovery: fetches the issuer's keys via
`/.well-known/openid-configuration` and its `jwks_uri`, caches and
refreshes them, and verifies RS256/384/512 and ES256/384/512 signatures
with the standard library's crypto packages only. Extracts RFC 7519,
Kubernetes ServiceAccount, and Substrate actor claims.

## Interface

- `oidcjwt.go:Verifier` - verifies JWTs from one trusted issuer; `oidcjwt.go:NewVerifier(issuer, audiences, source, opts...)` accepts a token when at least one audience matches.
- `oidcjwt.go:KeySource` - supplies public keys; called on first use, every `jwksRefreshInterval` (5m), and rate-limited on unknown key IDs.
- `oidcjwt.go:HTTPKeySource` - OIDC discovery via `oidcjwt.go:discoverKeysForIssuer`, `oidcjwt.go:keysFromJWKSet`, `oidcjwt.go:parseJWK`.
- `oidcjwt.go:Verify` - parses and verifies a JWT, returning `oidcjwt.go:Claims` (RFC 7519 + `oidcjwt.go:KubernetesClaims` + `oidcjwt.go:SubstrateClaims`).
- `oidcjwt.go:WithSubstrateClaims` - option making Verify decode `actor.ate.dev` / `worker-for-actor.ate.dev` claims (`oidcjwt.go:decodeSubstrateClaims`) and refuse a token carrying both.
- `oidcjwt.go:Refresh` - explicit key fetch; failures keep cached keys.
- `oidcjwt.go:ErrSigningKeysUnavailable` - wrapped by key-availability failures (retryable, distinct from token errors).
- `oidcjwt.go:verifySignature` - RS256/RS384/RS512 (PKCS1v15) and ES256/ES384/ES512 (P-256/384/521) only.
- `http.go:NewHTTPClient` - client for discovery/JWKS with an optional CA file (re-read per request via `http.go:rootsTransport`) and optional bearer-token file (`http.go:issuerDiscoveryTransport`).
- `http.go:NewHTTPClientWithRoots` - same transport against a caller-supplied pool loader.

## Invariants

- Nothing from the payload except `iss` is consulted before signature verification; algorithms and keys come only from the verified header and fetched keys (`oidcjwt.go:Verify` comments).
- Substrate claims are decoded only by a verifier built with `WithSubstrateClaims`; a token carrying both claim kinds, or a malformed one from the trusted issuer, is refused (`oidcjwt.go:decodeSubstrateClaims`).
- Made-up key IDs cannot each trigger a fetch: the first unknown-key fetch sets `lastUnknownKeyRefresh` before fetching, and repeats are rate-limited to `jwksRefreshRetryInterval` (1m) (`oidcjwt.go:key`, `oidcjwt.go:cachedKey`).
- A canceled fetch records `lastFailedRefresh` (starting the failure backoff) only when a cached key exists; with none cached it does not, so the next verify retries the fetch. Cached keys keep serving through refresh failures with a warning (`oidcjwt.go:key`).
- Unusable JWKs are skipped, not fatal, because selection is by kid and signatures are verified; an empty or fully unusable JWKS is an error (`oidcjwt.go:keysFromJWKSet`).
- A failed CA-pool reload keeps the last pool (`http.go:rootsTransport.current`); the discovery token file is read on every matching request so rotation is picked up (`http.go:issuerDiscoveryTransport.RoundTrip`).
- A discovery token file requires a CA file (`http.go:NewHTTPClient`).

## Side Effects

- HTTP GETs to the issuer's discovery document and JWKS URI (10s client timeout) (`oidcjwt.go:discoverKeysForIssuer`, `http.go:newHTTPClient`).
- Reads the CA file and token file from disk per request when configured (`http.go:rootsTransport`, `http.go:issuerDiscoveryTransport`).

## Verification

- Run: `go test ./internal/oidcjwt/...`
