# sink

The peers atunnel talks to in the benchmark: the ingress actor in each actor's
sandbox, behind ingress, and the fake egress gateway, behind egress. Both
answer with a fixed body and count what they see.

## Interface

- `gateway.go:Gateway` / `gateway.go:NewGateway` - stands in for the remote egress gateway: an mTLS server accepting CONNECT over HTTP/2 or HTTP/1.1, configured from `gateway.go:GatewayConfig` (`CredentialBundle`, `ClientCA`, `BodyBytes`).
- `gateway.go:Gateway.Serve` - serves TLS on a listener until ctx ends, with the real gateway's Envoy HTTP/2 settings (`MaxConcurrentStreams` 100, `MaxReadFrameSize` 16 KiB, 1 MiB receive buffers).
- `gateway.go:GatewayStats` / `gateway.go:Gateway.Stats` - counters: `Tunnels`, `TunnelsH2`, `TunnelsOpen`, `Connections`, `Rejected`, `Requests`, `BytesIn`.
- `gateway.go` port constants - `AnswerPort` ("80", HTTP requests answered with the fixed body) and `DiscardPort` ("9", raw bytes read and dropped).
- `gateway.go:AdminHandler` - serves the gateway's counters and process stats on `/stats` and `/healthz`.
- `ingress_actor.go:IngressActor` / `ingress_actor.go:NewIngressActor` - stands in for an actor's program receiving ingress: an HTTP server answering every request with the same body, over HTTP/1.1 and h2c.
- `ingress_actor.go:IngressActorStats` / `ingress_actor.go:IngressActor.Stats` - counters: `Requests`, `ConnectionsSeen`, `ConnectionsOpen`.
- `ingress_actor.go:IngressActor.Serve` - serves on a listener until ctx ends, counting connections through `ConnState`.

## Invariants

- The gateway accepts only CONNECT whose destination port is `AnswerPort` or `DiscardPort`; anything else is rejected and counted (`gateway.go:Gateway.serveHTTP`).
- The gateway never dials the named destination; it answers or discards by port (`gateway.go` header comment, `gateway.go:Gateway.serveHTTP`).
- An HTTP/2 tunnel is the stream itself (request body in, flushed response body out); an HTTP/1.1 tunnel hijacks the connection and writes `200 Connection Established` first (`gateway.go:Gateway.serveHTTP`).
- Both servers require client certificates signed by the configured CA (`gateway.go:NewGateway` sets `RequireAndVerifyClientCert`).
- `serveUntil` treats `http.ErrServerClosed` after ctx ends as success (`ingress_actor.go:serveUntil`).

## Side Effects

- Both peers listen on TCP (gateway TLS, ingress actor plain HTTP/1.1 and h2c) and mutate only their in-memory atomic counters (`gateway.go:Gateway.Serve`, `ingress_actor.go:IngressActor.Serve`).

## Verification

- Run: `go test ./benchmarking/atunnel/internal/sink/...` from the repo root.
