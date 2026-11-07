# dns

## Overview

Per-actor DNS relay for atunnel. It listens on UDP/53 and TCP/53 in the
sandbox's gateway network namespace, checks every query against the actor's
EgressPolicy, rewrites transaction IDs, forwards to the worker pod's upstream
resolvers with failover, and relays validated responses back to the actor.

## Interface

- `Relay` (`relay.go:Relay`): configured relay; listens in the sandbox gateway netns and dials upstreams from the worker netns.
- `NewRelay(resolvConfPath string) (*Relay, error)` (`relay.go:NewRelay`): builds a relay from a resolv.conf's nameservers.
- `NewRelayForUpstreams(upstreams []string) (*Relay, error)` (`relay.go:NewRelayForUpstreams`): builds a relay for ordered `host:port` upstreams.
- `(*Relay).Serve(ctx, ns netns.Handle, policy PolicySource) (*Server, error)` (`relay_linux.go:Serve`): Linux-only; binds the sockets and starts serving.
- `Server` (`server.go:Server`): one sandbox's running relay, returned by `Relay.Serve`.
- `(*Server).Stop(ctx context.Context) error` (`server.go:Stop`): cancels in-flight queries, closes sockets, waits for serving goroutines.
- `PolicySource` interface with `EgressPolicy(ctx) (*ateletpb.EgressPolicy, error)` (`policy.go:PolicySource`): supplies the policy each query is checked against.
- Subdirectories:
  - [freemap](./freemap/EGM.md) - 64k-entry free-slot bit-set used to allocate unique upstream transaction IDs.
  - [protocol](./protocol/EGM.md) - DNS wire-format helpers: sanity checks, transaction-ID rewriting, error packets, TCP framing.

## Invariants

- Fails closed: a nil `PolicySource` refuses every query (`server.go:serverConfig`, `policy.go:egressPolicyCheck`); a query policy fetch error denies the query (`policy.go:egressPolicyCheck`).
- A nil policy or one with no matching hostname rules allows nothing; invalid hostname patterns are ignored so remaining patterns still apply (`evaluator.go:newEvaluator`).
- Only OpCode QUERY with exactly one question is forwarded; AXFR and IXFR get NotImplemented, malformed queries get FormatError (`dnshandler.go:onRequestCommon`).
- Disallowed names get an NXDOMAIN reply whose root SOA has TTL 0 so the answer is not cached (`dnshandler.go:onRequestCommon`, `protocol/protocol.go:NXDomainPacket`).
- A UDP response is delivered only if its transaction ID, canonicalized question, and source upstream address match a pending entry (RFC 5452) (`dnshandler.go:onUDPResponse`).
- Each forwarded query gets a unique upstream transaction ID allocated from a random starting point; the actor's original ID is restored on delivery (`pendingreq.go:record`, `dnshandler.go:onUDPResponse`).
- Capacity limits: at most 64 in-flight UDP queries and 16 TCP connections; excess queries are dropped or refused (`limiter.go:maxInFlight`, `limiter.go:maxConnections`).
- `limiterSlots.release` must pair with a successful `tryAcquire`; an unbalanced release panics (`limiter.go:release`).
- SERVFAIL/NOTIMP/REFUSED responses fail over to the next upstream while one remains, otherwise the deferred error response is returned (`dnshandler.go:isFailoverRCode`, `pendingreq.go:sweep`).
- `NewRelayForUpstreams` requires at least one resolvable upstream (`relay.go:NewRelayForUpstreams`); a resolv.conf naming no usable nameserver is an error (`resolvconf.go:resolvConfNameservers`).
- TCP streams are closed after 30s idle (`tcphandler.go:defaultConnectionTimeout`).

## Side Effects

- Binds UDP and TCP port 53 inside the sandbox gateway network namespace via `netns.Do`, and opens a worker-side UDP egress socket (`relay_linux.go:Serve`, `server.go:newServer`).
- Sends and receives DNS traffic on the network; rewrites the transaction ID of query packets in place and clones them for failover retries (`pendingreq.go:record`).
- Runs serving goroutines and a 50ms sweep ticker that resends or evicts timed-out queries (`server.go:newServer`, `udphandler.go:sweepLoop`).
- `Server.Stop` cancels the serve context and closes the UDP sockets and TCP listener (`server.go:Stop`).
- Reads the worker's resolv.conf from disk (`resolvconf.go:resolvConfNameservers`).
- Emits slog log lines for configuration, socket errors, and denials.

## Verification

- Run: `go test ./internal/atunnel/dns/...`
