# credprovider/v1

## Overview

The `substrate.credprovider.v1` proto package defines the plugin API a secret
backend implements so Substrate can fetch an external credential without
storing it. `credprovider.proto` is the source of truth; the two `*.pb.go`
files are generated. Go package: `go.ate.dev/substrate/api/credprovider/v1`
(package `credprovider`). Plugin implementations live in their own repository
under the `agent-substrate-plugins` org, except the in-tree
`cmd/credential-provider/kubernetes-secrets`.

## Interface

- `service CredentialProvider` (`credprovider.proto:25`): one RPC,
  `FetchSecret(FetchSecretRequest) returns (FetchSecretResponse)`, which
  resolves a credential URI to secret material.
- `FetchSecretRequest` (`credprovider.proto:32`): `uri` in the form
  `ate-secret://<provider name>/<provider tail>` plus the verified
  `actor_spiffe_id` of the actor on whose behalf the secret is fetched.
- `FetchSecretResponse` (`credprovider.proto:44`): `opaque_bytes`, the raw
  secret bytes, marked `debug_redact`.
- Generated Go stubs: `CredentialProviderClient`, `CredentialProviderServer`,
  `RegisterCredentialProviderServer` (`credprovider_grpc.pb.go:47`, `:79`,
  `:106`), and the message structs in `credprovider.pb.go`.

## Invariants

- `FetchSecretRequest.actor_spiffe_id` always contains a SPIFFE ID, verified
  by the egress gateway before the call (`credprovider.proto:38`-`:40`).
- Authorization of the request context is the provider's to apply; Substrate
  does not store the credential (`credprovider.proto:21`-`:24`,
  `:26`-`:28`).
- `opaque_bytes` is redacted from debug output (`debug_redact` on
  `credprovider.proto:46`).

## Side Effects

- `RegisterCredentialProviderServer` attaches the handler to a
  `grpc.ServiceRegistrar` (`credprovider_grpc.pb.go:106`); the package is
  otherwise pure type definitions.

## Verification

- Run: `go test ./api/...`
