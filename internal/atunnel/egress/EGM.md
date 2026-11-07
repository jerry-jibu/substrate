# egress

## Overview

Worker-side actor egress for atunnel. `Proxy` accepts transparently intercepted
actor TCP connections and forwards them through an mTLS CONNECT tunnel to the
egress gateway. `Client` opens those tunnels (HTTP/2 or HTTP/1.1 CONNECT), and
`BrokerCertificateSource` mints and renews the short-lived actor certificate
from the node-local atelet credential broker.

## Interface

- `Proxy` (`egress.go:Proxy`): activation-aware egress proxy, long-lived across actor activations.
- `New(originalDestination func(net.Conn) (string, error)) (*Proxy, error)` (`egress.go:New`): creates the proxy.
- `(*Proxy).Bind(actorUID string) (func(context.Context, net.Listener) error, error)` (`egress.go:Bind`): returns a listener serve function bound to the actor's activation.
- `(*Proxy).Activate(actorUID string, dialer tunnelDialer, certificateSource actorCertificateSource, expiresAt time.Time) error` (`egress.go:Activate`): enables egress and certificate renewal for the actor.
- `(*Proxy).Deactivate(ctx context.Context, actorUID string) error` (`egress.go:Deactivate`): disables egress, closes and drains the actor's streams.
- `PortFromListenAddr(listenAddress string) (uint16, error)` (`egress.go:PortFromListenAddr`): the port each sandbox redirect aims at; the address is never bound.
- `ClientConfig` (`client.go:ClientConfig`): gateway address, TLS server name, client-certificate callback, trust bundle path.
- `Client` / `NewClient(cfg ClientConfig, opts ...clientOption) (*Client, error)` (`client.go:NewClient`): CONNECT client; implements `tunnelDialer` (`client.go:tunnelDialer` assertion).
- `(*Client).DialContext(ctx, destination string) (net.Conn, error)` (`client.go:DialContext`): opens a CONNECT tunnel to an `IP:port` destination.
- `(*Client).Close() error` (`client.go:Close`): closes pooled connections and cancels in-flight dials.
- `ErrGatewayHandshake` (`client.go:ErrGatewayHandshake`) and `ConnectRejectedError` (`client.go:ConnectRejectedError`): sentinel errors for TLS-refused and non-2xx CONNECT answers.
- `BrokerConfig` (`credential.go:BrokerConfig`) and `NewBrokerCertificateSource(cfg BrokerConfig) (*BrokerCertificateSource, error)` (`credential.go:NewBrokerCertificateSource`): client for the atelet credential broker.
- `(*BrokerCertificateSource).MintWorkerCertificate(ctx) (time.Time, error)` (`credential.go:MintWorkerCertificate`): mints a certificate for the actor key, returns its expiry.
- `(*BrokerCertificateSource).GetClientCertificate(*tls.CertificateRequestInfo) (*tls.Certificate, error)` (`credential.go:GetClientCertificate`): supplies the current certificate to the TLS handshake.

## Invariants

- `Activate` requires a non-empty actor UID, a dialer, a certificate source, and a future expiry; activating twice fails (`egress.go:Activate`).
- Renewal is scheduled at 90% of the credential's remaining lifetime (`egress.go:renewAfter`); a denied renewal (gRPC Aborted, FailedPrecondition, PermissionDenied) permanently blocks new tunnels (`egress.go:renew`).
- Expiry blocks only new tunnels; connections admitted with a valid certificate drain normally (`egress.go:handle`).
- Renewal can never restore a credential after `Deactivate` cleared it; the check happens under the same mutex (`egress.go:renew`).
- Tunnel opens are bounded by a 10s timeout so a queued CONNECT frees the gateway's connection cap (`egress.go:tunnelOpenTimeout`).
- `NewClient` validates the gateway address, server name, certificate callback, and a trust bundle that parses to at least one certificate; TLS is at least 1.2 with ALPN h2/http/1.1 (`client.go:NewClient`).
- Tunnel destinations must be an IP host and a port in 1-65535 (`client.go:validateDestination`).
- At most one gateway connection dial runs at a time (`client.go:acquireConn`); HTTP/2 connections are retired once the client certificate changes, since the gateway authenticates each CONNECT with the handshake certificate (`client.go:acquireConn`).
- The actor private key is generated once per activation and never leaves atunnel; only its CSR crosses the broker socket (`credential.go:NewBrokerCertificateSource`).
- A minted chain must match the private key, have a valid lifetime, and include ExtKeyUsageClientAuth, or it is rejected (`credential.go:mintOnce`); `Unavailable` mints are retried at most 4 times (`credential.go:mintAttempts`).
- `GetClientCertificate` refuses to present an expired certificate (`credential.go:GetClientCertificate`).

## Side Effects

- Dials the egress gateway with mTLS, sends CONNECT requests, and relays actor bytes both directions (`egress.go:handle`, `client.go:DialContext`).
- Runs goroutines per activation for certificate renewal and per accepted connection for forwarding (`egress.go:Activate`, `egress.go:handle`); `Deactivate` cancels them, waits, and closes the dialer (`egress.go:Deactivate`).
- Generates an ECDSA P-256 actor key (`credential.go:NewBrokerCertificateSource`).
- Dials the atelet Unix socket and calls `MintActorCertificate` over gRPC (`credential.go:mintOnce`).
- Reads the trust bundle from disk at client construction (`client.go:NewClient`); reads the worker credential bundle for broker TLS (`credential.go:NewBrokerCertificateSource` via `ateletdial.TLSConfig`).
- Copies streams using pooled 32 KiB buffers to keep idle memory low (`streamcopy.go:streamCopy`).
- Emits slog warnings for expired or denied certificates and failed tunnels.

## Verification

- Run: `go test ./internal/atunnel/egress/...`
