# credbundle

## Overview
Parses and caches PEM credential bundles and trust pools as written by the Kubernetes Pod Certificates mechanism, with re-read on file change so certificate rotations are picked up at the next TLS handshake.

## Interface
- `credbundle.go:Loader` - returns a `GetCertificate` func for a `tls.Config` from a bundle path.
- `credbundle.go:ClientLoader` - client-side counterpart, suitable as `GetClientCertificate`.
- `credbundle.go:PoolLoader` - returns a func for `GetConfigForClient` producing a fresh `*x509.CertPool` per connection.
- `credbundle.go:SystemPoolLoader` - `PoolLoader` for a pool that also holds the system roots.
- `credbundle.go:Parse` - reads a private key and certificate chain from a bundle file.
- `credbundle.go:ParsePool` - reads a PEM trust-bundle file into an `*x509.CertPool`.

## Invariants
- The parsed bundle is cached: each handshake stats the file and re-reads only when it changed, so rotations are picked up without paying the read and parse cost when nothing changed (`credbundle.go:Loader`).
- A pool built once at startup never sees a CA rotation; the pool func must be called per connection to keep verification current (`credbundle.go:PoolLoader`).
- `ParsePool` errors when the file holds no certificates: an empty trust pool would silently reject every peer (`credbundle.go:ParsePool`).

## Side Effects
- Stats and reads bundle/trust files from disk on each handshake or call (`credbundle.go:Loader`, `credbundle.go:PoolLoader`).

## Verification
- Run: `go test ./internal/credbundle/...`
