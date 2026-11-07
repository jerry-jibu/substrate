# sdsmint

## Overview

Implements `atenet sdsmint`, a minting SDS (Secret Discovery Service)
server: Envoy subscribes by SNI name over delta xDS and receives a freshly
signed TLS leaf plus its private key. The MITM CA is loaded from a localca
pool JSON and never leaves this process.

## Interface

- `cmd.go:NewSdsmintCmd` - the cobra command with flags `--uds-path`, `--ca-pool-path`, `--leaf-cert-ttl`, `--log-level`.
- `run.go:run` (unexported) - validates config, loads the CA pool (`localca.NewRefreshingPool`), builds the minter, serves gRPC on the unix socket until SIGINT/SIGTERM.
- `minter.go:minter`, `minter.go:MintedCert` - mint per-hostname leaves (PEM chain plus private key, notAfter, hex serial for the issuance audit log).
- `server.go:server` - implements Envoy's Secret Discovery Service; `DeltaSecrets` is the only method implemented.
- `deltastream.go:(*server).DeltaSecrets` - the stateful delta-SDS loop (subscribe, withdraw, bare ack, NACK handling).
- `listen.go:listen` (unexported) - binds the SDS unix socket.

## Invariants

- The only transport is a unix domain socket, chmod 0600, because leaf private keys transit it; there is no TCP alternative (`cmd.go` `--uds-path` help, `listen.go:listen`).
- State-of-the-world SDS (`StreamSecrets`, `FetchSecrets`) is deliberately left unimplemented so an Envoy configured with anything other than DELTA_GRPC fails immediately and visibly (`server.go:server`).
- `--leaf-cert-ttl` is refused outside 2 minutes to 24 hours: below the band the 5-minute back-dating for clock skew dominates, above it a steadily used name would carry the same MITM leaf too long (`cmd.go:validateTTL`).
- The xDS resource TTL is derived at half the leaf lifetime, so an actively used name is re-minted about twice per lifetime and an idle one is dropped (`cmd.go` `--leaf-cert-ttl` help).
- Hostnames are validated by `checkHostSyntax`; a refused name is denied and withdrawn, never minted (`minter.go:certificate`, `minter.go:errHostNotAllowed`).
- A literal IP in the SNI position lands in `IPAddresses`, not `DNSNames`, or clients reject the leaf (`minter.go:certificate`).
- Every response carries a nonce from one atomic server-wide counter, which is what makes later requests recognizable as ACK/NACK (`server.go:server.nonce`).
- The server never pushes unprompted; a bare ack sends nothing, and unsubscribe is ignored (`deltastream.go` delta stream handling).

## Side Effects

- Signs certificates with the localca pool (held in-process; the CA stays off the data plane).
- Opens and removes a unix socket file; serves a long-lived gRPC delta stream per Envoy client; writes an issuance audit log line per certificate.

## Verification

- Run: `go test ./cmd/atenet/internal/sdsmint/...`
