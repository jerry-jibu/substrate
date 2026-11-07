# ateapiauth

## Overview
gRPC dial options for clients talking to ateapi with mutual TLS, built from a PEM credential bundle and a CA file.

## Interface
- `client.go:ClientConfig` - fields for target, CA file, and client credential bundle (leaf certificate plus PKCS8 key).
- `client.go:DialOptions` - returns the `grpc.DialOption` set described by the config, for `grpc.NewClient`.
- `client.go:DefaultServiceAccountCAFile` - default CA path, the Kubernetes service-account CA mount.

## Invariants
- The credential bundle is re-read on every handshake, so in-place pod-certificate rotations are picked up without redialing (`client.go:ClientConfig`).
- Both the CA file and the client credential are required; `DialOptions` errors when either is missing (`client.go:DialOptions`).
- Dialing uses round-robin load balancing across every resolved address (`client.go:roundRobinServiceConfig`).

## Side Effects
- Reads the CA file and credential bundle from disk at handshake time, through `internal/credbundle` (`client.go:DialOptions`).

## Verification
- Run: `go test ./internal/ateapiauth/...`
