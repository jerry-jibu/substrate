# objectstoreexchangepb

## Overview
Generated protobuf and gRPC code for the object-store credential exchange (`exchange.proto`): the `objectstoreexchange.CredentialExchange` service served by the control plugin sidecar and dialed by node plugins to trade an actor JWT for a scoped storage credential.

## Interface
- `objectstoreexchange.CredentialExchange` service method (`exchange_grpc.pb.go:MethodName`): ExchangeActorJwt.
- Client and server stubs: `exchange_grpc.pb.go` (`NewCredentialExchangeClient`, `CredentialExchangeServer`); message types in `exchange.pb.go` (`ExchangeActorJwtRequest`, `ExchangeActorJwtResponse`).
- Consumers: `internal/objectstoreplugin/exchange` (server) and `internal/objectstoreplugin/exchangeclient` (client).

## Invariants
- Regenerated code only: edit `exchange.proto` and regenerate, never the `.go` files.

## Side Effects
- ["None"] (generated message and service definitions only).

## Verification
- Run: `go test ./internal/proto/objectstoreexchangepb/...` (generated-code build check).
