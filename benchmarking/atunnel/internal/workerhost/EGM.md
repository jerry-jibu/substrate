# workerhost

Hosts atunnel the way an ateworker does, with the worker's own code:
`atunnel.Start` once, then per actor the calls `cmd/ateworker-gvisor` makes, in
its order. Only the sandbox itself is missing: each actor's network namespaces,
veth, nftables redirect, egress listener, and DNS relay are real, but no gVisor
runs in them. Linux only (`//go:build linux`).

## Interface

- `workerhost.go:Start` - starts atunnel from `cfg` (flags registered by `atunnel.RegisterFlags`) and hosts `actors` actors in atespace `Atespace` ("bench"), each egress tunneling to `gatewayAddress`.
- `workerhost.go:Host.Info` / `workerhost.go:Info` / `workerhost.go:Sandbox` - the hosted actors: `Actor` (atespace/name, the routing-header value), `UID`, `NetNS` (namespace path), and `ActorPort` (80, `actornet.ActorHTTPUpstream`), served on the tunnel's `/stats`.
- `workerhost.go:Host.Close` - tears every actor down as a worker does: deactivate its traffic, then remove its network.

## Invariants

- `host` follows `cmd/ateworker-gvisor`'s `RunWorkload` order per actor: `Deactivate` any previous activation, `PrepareEgress` (minting the actor certificate through atunnel's atelet client), `actornet.ServeSandbox`, `ActivateEgress`, then `ActivateIngress` (`workerhost.go:Host.host`).
- Actor names are `actor-<i>` in atespace `bench` with deterministic UUIDs (`workerhost.go:Start`).
- A hosting failure closes what was already started, joining the errors (`workerhost.go:Start`).

## Side Effects

- Creates real network namespaces, veth pairs, and nftables redirects per actor through `actornet.ServeSandbox`, and drives the real atunnel process (`internal/atunnel`) (`workerhost.go:Host.host`).
- Mints actor certificates through atunnel's egress preparation (`workerhost.go:Host.host`).

## Verification

- Run: `go test ./benchmarking/atunnel/internal/workerhost/...` from the repo root.
