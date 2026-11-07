# actornet

## Overview

Linux-only package that builds and tears down one actor's private point-to-point network inside a worker. It creates named network namespaces (`netns.CreateNamed`), a veth pair or microVM tap on the fixed `169.254.17.0/30` subnet, nftables rules that reject link-local egress and redirect all other TCP to the atunnel egress port, and it writes the sandbox's `/etc/resolv.conf` and `/etc/hosts` into the actor rootfs. Used by both the gVisor and microVM worker runtimes.

## Interface

- `sandbox.go:SetupSandboxNetwork(ctx, SandboxNetworkConfig) (*SandboxNetwork, error)` - creates the actor runtime namespace plus (for gVisor, `cfg.Veth`) a separate gateway namespace with the veth pair, and installs egress rules.
- `sandbox.go:SandboxNetworkConfig` - per-actor inputs: `ActorUID`, `Veth`, `EgressPort`, `DNSPolicy dns.PolicySource`.
- `sandbox.go:SandboxNetwork` - returned handles: `ActorUID`, `RuntimeNetNS`, `GatewayNetNS`.
- `sandbox.go:ServeSandbox(ctx, cfg, egress egressServer, resolver *dns.Relay) (*SandboxSession, error)` - builds the network and serves DNS and redirected egress from the gateway namespace; nil server or resolver leaves that part unserved (fails closed).
- `sandbox.go:SandboxSession.Close(ctx)` - idempotent teardown: closes sockets, stops DNS, waits for serving goroutines, removes namespaces.
- `sandbox.go:SandboxSession.Dialer()` / `sandbox.go:SessionHolder` - dial into the sandbox from the gateway namespace; `SessionHolder.Replace/Session/Close/Dialer` manage the one session a worker serves.
- `sandbox.go:SandboxGatewayNetNSName(actorUID)` and `sandbox.go:CleanupSandboxNetwork(network)` - gateway namespace naming and cleanup.
- `net.go` constants: `ActorVethName` ("eth0"), `ActorVethGateway` ("169.254.17.1"), `ActorVethIP` ("169.254.17.2"), `ActorVethSubnet` ("169.254.17.0/30"), `ActorHTTPUpstream`; helpers `MustParseAddr`, `MustParseIP`, `MustParseMAC`.
- `resolvconf.go:SandboxResolvConf(nameserver, podResolvConf)` - rewrites nameserver lines while keeping the pod's search domains and options.
- `resolvconf.go:SandboxHosts(hostname, ip)`; `resolvconf.go:WriteRootfsResolvConf/WriteRootfsHosts` - write `/etc` files inside a rootfs via `os.Root`.
- `etcfiles.go:WriteRootfsEtcFiles(rootfs, hostname)` - reads the worker pod's `/etc/resolv.conf` and installs both files.

## Invariants

- `sandbox.go:SetupSandboxNetwork` requires a non-empty actor UID and a non-zero `EgressPort`; the actor's own `/30` is excluded from redirect and link-local rejection so ingress replies and DNS reach local sockets (`sandbox.go:installEgressRules`).
- The `forward` chain in both nftables tables has policy drop, backing the forwarding sysctls; nothing in a sandbox namespace is routed (`sandbox.go:installEgressRules`, `sandbox.go:sandboxSysctls`).
- Sysctls are pinned before any link exists and in a fixed order (forwarding before `accept_redirects`); a link copies conf/default at creation (`sandbox.go:prepareNamespace`).
- Egress binding happens before serving, so a connection cannot pick up a later activation's credentials (`sandbox.go:serveSandboxEgress`).
- `SessionHolder.Replace` closes the previous session before installing the next; overwriting it would leak namespaces, `/run/netns` mounts, and goroutines (`sandbox.go:SessionHolder.Replace`).
- A nil `DNSPolicy` refuses every DNS query (`sandbox.go:SandboxNetworkConfig`).
- `resolvconf.go:writeRootfsEtcFile` refuses empty content, creates `etc/` if missing, unlinks the target first (no writes through existing links), and opens with `O_EXCL` inside `os.Root`.

## Side Effects

- Creates and removes named network namespaces under `/run/netns` (`netns.CreateNamed`/`netns.RemoveNamed` in `sandbox.go`).
- Mutates netlink state: veth pairs, addresses, routes, link up (`sandbox.go:setupVethPair`).
- Programs nftables tables `worker-actor` (IPv4) and `worker-actor-v6` (IPv6) in the gateway namespace (`sandbox.go:installEgressRules`).
- Sets per-namespace sysctls (unprivileged port start, forwarding, redirects, rp_filter, route_localnet) (`sandbox.go:sandboxSysctls`).
- Writes `/etc/resolv.conf` and `/etc/hosts` into the actor rootfs; reads the worker pod's `/etc/resolv.conf` (`etcfiles.go:WriteRootfsEtcFiles`, `resolvconf.go:writeRootfsEtcFile`).
- Spawns goroutines serving egress listeners and DNS (`sandbox.go:ServeSandbox`); closes listeners and waits on `Close`.

## Verification

- Run: `go test ./internal/ateworker/actornet/...`
