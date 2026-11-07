# fakeatelet

Serves the part of atelet's WorkerSupport API that atunnel calls, so atunnel's
own client code talks to a fake unchanged. It listens on the node-local socket
with atelet's certificate, mTLS against the benchmark CA.

## Interface

- `fakeatelet.go:Server` - the fake atelet; embeds `ateletpb.UnimplementedWorkerSupportServer` and holds the CA certificate and key.
- `fakeatelet.go:New` - loads the CA from `pkiDir` via `pki.LoadCA`.
- `fakeatelet.go:Server.Serve` - listens on `socketPath` (atelet's `AteletBundle` serving certificate, `RequireAndVerifyClientCert` against `CAFile`, TLS 1.3) and serves the gRPC `WorkerSupport` service until ctx ends.
- `fakeatelet.go:Server.MintActorCertificate` - verifies the CSR, signs it as the actor's identity (SPIFFE URI `spiffe://substrate-actor.local/worker/actor/<atespace>/<name>`, client-auth EKU, `actorCertificateLifetime` of 24h), and returns it in a `MintActorCertificateResponse`.
- `fakeatelet.go:Server.GetActorEgressPolicy` - always reports no policy; only the DNS relay asks, and the benchmark's egress targets addresses, not names.

## Invariants

- Every served RPC keeps atelet's wire contract: `Serve` registers the server as `ateletpb.RegisterWorkerSupportServer`, and unimplemented methods come from the embedded `UnimplementedWorkerSupportServer` (`fakeatelet.go:Server.Serve`).
- Minted actor certificates carry the actor identity from the request via `substratex509.AddActorIdentityToCertificate` and last `actorCertificateLifetime` (24h), longer than any run since atunnel renews at 90% (`fakeatelet.go:Server.MintActorCertificate`).
- A CSR that fails `x509.ParseCertificateRequest` or `CheckSignature` is rejected with an error (`fakeatelet.go:Server.MintActorCertificate`).

## Side Effects

- `Serve` creates the socket's parent directory, removes a stale socket file, and listens on a Unix socket at `socketPath` (`fakeatelet.go:Server.Serve`).
- `MintActorCertificate` signs certificates with the CA key loaded from `pkiDir` (`fakeatelet.go:Server.MintActorCertificate`).

## Verification

- Run: `go test ./benchmarking/atunnel/internal/fakeatelet/...` from the repo root.
