# exchangeclient

## Overview
The GCS node plugin's side of the credential exchange ("Package exchangeclient", `exchangeclient.go`). For each transfer it trades the actor JWT atelet handed it for a scoped GCS access token, caches the token and the storage client built with it under that JWT, and hands the client to the transfer. This binary's counterpart to `exchange`; a provider's own plugin brings its own client.

## Interface
- `exchangeclient.go:New(cfg Config) (*Source, error)` builds the credential `Source` used by `objectstoreplugin.Credentials`.
- `exchangeclient.go:Config` carries the exchange endpoint and TLS settings; `exchangeclient.go:Source` implements the per-transfer client lookup.

## Invariants
- The token and its storage client are cached under the actor JWT they were issued for; a different JWT gets its own entry ("Package exchangeclient" doc).

## Side Effects
- Calls the credential exchange gRPC service (`objectstoreexchange.CredentialExchange`) on the control sidecar.
- Builds GCS storage clients from the issued tokens and keeps them in memory.

## Verification
- Run: `go test ./internal/objectstoreplugin/exchangeclient/...`
