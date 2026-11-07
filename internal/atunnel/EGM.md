# atunnel

## Overview

Wires the tunnel into a worker: its flags, listeners, DNS relay, and per-actor
ingress and egress. Both worker runtimes (gVisor and microVM) build their
tunnel here so they cannot drift. The package is Linux-only (`tunnel.go`
`//go:build linux`).

## Interface

- `FlagSet` (`tunnel.go:FlagSet`): flag registration surface shared by flag.FlagSet and pflag.FlagSet.
- `Config` (`tunnel.go:Config`) / `RegisterFlags(fs FlagSet) *Config` (`tunnel.go:RegisterFlags`): flags `atunnel-listen-address` (:443), `atunnel-connect-listen-address` (:8443), `atunnel-credential-bundle`, `atunnel-trust-bundle`, `atunnel-client-identity`, `atunnel-broker-identity`, `atunnel-egress-listen-address` (0.0.0.0:15001), `atunnel-egress-trust-bundle`, and `dns-egress-policy`.
- `Tunnel` (`tunnel.go:Tunnel`): state one worker shares across its actors (Ingress, Egress, EgressPort, DNSRelay).
- `Start(ctx, cfg Config, upstream string) (*Tunnel, error)` (`tunnel.go:Start`): builds the DNS relay and starts serving actor ingress.
- `ActorEgress` (`tunnel.go:ActorEgress`) / `(*Tunnel).PrepareEgress(ctx, actor, gateway) (*ActorEgress, error)` (`tunnel.go:PrepareEgress`): mints the actor certificate and builds its gateway client; nil gateway yields nil.
- `(*Tunnel).ActivateEgress(actor, actorEgress) error` (`tunnel.go:ActivateEgress`): starts tunneling the actor's outbound traffic.
- `(*Tunnel).ActivateIngress(actor, dial) error` (`tunnel.go:ActivateIngress`): admits the actor's inbound traffic through `dial`.
- `(*Tunnel).Deactivate(ctx, actor) error` (`tunnel.go:Deactivate`): stops both directions and drains active streams.
- `(*Tunnel).ActorEgressPolicy(actor) dns.PolicySource` (`egresspolicy.go:ActorEgressPolicy`): cached source of the actor's egress policy for its DNS relay.
- Subdirectories:
  - [dns](./dns/EGM.md) - per-actor DNS relay with policy enforcement, transaction-ID rewriting, and upstream failover.
  - [egress](./egress/EGM.md) - actor egress proxy, mTLS CONNECT client, and atelet certificate broker client.
  - [ingress](./ingress/EGM.md) - mTLS ingress reverse proxy and CONNECT tunnel endpoint for active actors.

## Invariants

- Both worker binaries build their tunnel from this package so their setups cannot diverge (`tunnel.go` package comment).
- `Start` rejects an invalid `dns-egress-policy` mode (`tunnel.go:Start`, `egresspolicy.ValidateDNSEgressPolicy`).
- `PrepareEgress` mints the actor certificate before the workload starts, so configured tunneled egress fails closed; a nil gateway means the actor has no tunneled egress (`tunnel.go:PrepareEgress`).
- The egress policy source serves one fetch per TTL window (5s) shared via singleflight, bounded by a 2s fetch timeout (`egresspolicy.go:EgressPolicy`, `egresspolicy.go:egressPolicyTTL`).
- Fail static: once a policy has been fetched, a failed refresh keeps serving the stale cached policy; before any fetch succeeds, a failure refuses the actor's DNS queries (`egresspolicy.go:record`).
- Callers must not modify the returned policy; it is shared across concurrent calls (`egresspolicy.go:EgressPolicy` doc).
- The first fetch failure is logged at error level; later stale-cache warnings are rate-limited to one per 5 minutes (`egresspolicy.go:egressPolicyStaleWarnInterval`).
- A listener that fails while serving exits the process via `serverboot.Fatal` (`tunnel.go:serve`).
- `Deactivate` attempts both ingress and egress teardown even when one fails (`tunnel.go:Deactivate`).

## Side Effects

- Listens on TCP for actor ingress HTTPS and mTLS CONNECT; the egress listen address is never bound, only its port is used as the sandbox redirect target (`tunnel.go:serve`, `egress.go:PortFromListenAddr`).
- Dials the node-local atelet worker-support Unix socket (`nodepath.WorkerSupportSocket`) over mTLS for certificate minting and egress-policy fetches (`egresspolicy.go:ActorEgressPolicy`, `tunnel.go:PrepareEgress`).
- Reads the worker pod's `/etc/resolv.conf` for upstream resolvers (`tunnel.go:resolvConfPath`).
- Caches each actor's egress policy in memory across queries (`egresspolicy.go:actorEgressPolicy`).
- Runs serving goroutines per listener and can terminate the process on serve failure (`tunnel.go:serve`).
- Emits slog logs with actor attribution attributes (`egresspolicy.go:ActorEgressPolicy`).

## Verification

- Run: `go test ./internal/atunnel/...`
