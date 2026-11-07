# apiconfig

## Overview
Loads and validates ateapi configuration: JWT authentication configuration from YAML/JSON, and the protobuf `APIConfig` from `.pb`, `.textproto`, or `.textpb` files.

## Interface
- `authn.go:AuthenticationConfig`, `authn.go:JWTProviderConfig` - trusted OIDC issuers, audiences, optional CA file and discovery token file.
- `authn.go:ParseAuthenticationConfig` - strictly parses and validates authentication config bytes.
- `authn.go:LoadAuthenticationConfig` - reads and strictly parses an authentication config file.
- `authn.go:ValidateAuthenticationConfig` - validates fields that do not require I/O.
- `loader.go:ParseTextProto` - strictly parses prototext bytes into an `ateconfigpb.APIConfig` and validates it.
- `loader.go:LoadConfig` - reads and strictly parses an `APIConfig` from a file, by extension.

## Invariants
- Parsing is strict: unknown fields are rejected rather than ignored (`authn.go:ParseAuthenticationConfig`, `loader.go:ParseTextProto`).
- `ParseTextProto` validates the parsed config's settings, including the log level (`loader.go:ParseTextProto`, `loader.go:validateConfig`).

## Side Effects
- `LoadAuthenticationConfig` and `LoadConfig` read files from disk (`authn.go:LoadAuthenticationConfig`, `loader.go:LoadConfig`).

## Verification
- Run: `go test ./internal/apiconfig/...`
