# ate-idp-server/internal

## Overview
Internal packages of the `ate-idp-server` binary. Holds the HTTP server that publishes the actor JWT issuer's OIDC discovery document and JWK set.

## Subdirectories & Boundaries
- server: [./server/EGM.md](./server/EGM.md) - serves `/.well-known/openid-configuration`, `/openid/v1/jwks`, and `/healthz` from a `KeySource`.
