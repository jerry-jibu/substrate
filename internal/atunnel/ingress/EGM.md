# ingress

## Overview

Worker-side actor ingress for atunnel. `Server` terminates mTLS from the
ingress router, routes each request to the currently active actor by name
through that actor's namespace-specific dialer, and also serves a bidirectional
CONNECT tunnel endpoint. Ordinary requests are reverse-proxied; gRPC traffic
goes to the actor as prior-knowledge h2c.

## Interface

- `DefaultConnectPort = 8443` (`ingress.go:DefaultConnectPort`): worker port for inbound mTLS CONNECT tunnels.
- `TargetPortHeader` (`ingress.go:TargetPortHeader`): header carrying the actor-side port; stripped before the request reaches the actor.
- `ParsePort(s string) (int, bool)` (`ingress.go:ParsePort`): TCP port in 1-65535 or ok=false.
- `DialFunc` (`ingress.go:DialFunc`): dialer matching `net.Dialer.DialContext`, scoped to the actor's namespace.
- `Config` (`ingress.go:Config`): credential bundle path, trust bundle path, allowed client identity, fixed upstream URL.
- `Server` / `NewServer(cfg Config) (*Server, error)` (`ingress.go:NewServer`): HTTPS reverse proxy for the worker's active actors; implements `http.Handler` (`ingress.go:ServeHTTP`).
- `(*Server).Serve(ctx, lis net.Listener) error` (`ingress.go:Serve`): serves request-proxy HTTPS.
- `(*Server).ServeConnect(ctx, lis net.Listener) error` (`ingress.go:ServeConnect`): serves the mTLS CONNECT tunnel endpoint.
- `(*Server).Activate(atespace, actorName, actorUID string, dial DialFunc) error` (`ingress.go:Activate`): routes the actor's requests through its dialer.
- `(*Server).Deactivate(ctx, atespace, actorName, actorUID string) error` (`ingress.go:Deactivate`): rejects new requests and drains the actor's handlers.

## Invariants

- The TLS front door requires a client certificate whose URIs include the configured `AllowedClientID` SPIFFE ID (`ingress.go:NewServer` `verifyConnection`).
- The trust bundle and credential bundle are reloaded per connection, so kubelet-projected CA and certificate rotation take effect without a pod restart (`ingress.go:GetConfigForClient`, `ingress.go:loadCredentialBundle`).
- Every request must carry exactly one `atenet.TargetActorHeader` naming an active actor; otherwise it is answered 421 with `X-Ate-Assignment-Stale` set (`ingress.go:authorize`, `ingress.go:sendRejectResponse`).
- Actor responses cannot spoof routing rejections: the proxy strips `X-Ate-Assignment-Stale` from upstream responses (`ingress.go:newActorProxy` `ModifyResponse`).
- `Activate` validates atespace and actor names, replaces an older incarnation under the same name, and rejects double activation of the same UID (`ingress.go:Activate`); `Deactivate` leaves a newer incarnation under the same name alone (`ingress.go:Deactivate`).
- Requests are canceled with `errActorDeactivated` when their actor is deactivated (`ingress.go:authorize`).
- CONNECT requires the CONNECT method and an authority with a valid port, and dials the actor with a 5s timeout (`ingress.go:serveConnectHTTP`).
- gRPC requests (HTTP/2 POST with `application/grpc` content type) are forwarded as prior-knowledge h2c; all other traffic is translated to HTTP/1.1 (`ingress.go:isGRPC`, `ingress.go:protocolMirrorTransport`).
- `NewServer` fails on a missing or malformed credential bundle or trust bundle so a bad projection fails the pod promptly (`ingress.go:NewServer`).

## Side Effects

- Serves HTTPS with `RequireAndVerifyClientCert` on the given listener; hijacks HTTP/1.1 CONNECT connections and relays tunnel bytes with half-close semantics (`ingress.go:serve`, `ingress.go:serveH1Connect`, `ingress.go:relayWithHalfClose`).
- Dials actors through per-activation dialers and keeps per-activation connection pools, closing idle ones on replace and deactivate (`ingress.go:newActorProxy`, `ingress.go:closeIdleUpstreamConnections`).
- Reads the credential bundle and trust bundle from disk, both at startup and per connection (`ingress.go:loadCredentialBundle`, `credbundle.PoolLoader`).
- Runs goroutines per activation and per relayed tunnel; contexts are canceled on deactivate and server shutdown (`ingress.go:Activate`, `ingress.go:serve`).
- Emits slog logs for rejected CONNECTs, upstream failures, and tunnel teardown.

## Verification

- Run: `go test ./internal/atunnel/ingress/...`
