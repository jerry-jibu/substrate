# netns

## Overview

Linux network namespace helpers: create named netns, run code or open
sockets inside one, dial IP literals from inside one, and write per-netns
sysctls. Entering a namespace is a property of the OS thread, so every
entry locks a thread and restores the caller's namespace before returning.

## Interface

- `netns.go:Handle` - alias for `vishnetns.NsHandle`, the descriptor callers pass around.
- `netns.go:GetFromName` - opens a named netns under `/run/netns`; `netns.go:validateName` rejects empty, `.`, `..`, and any name containing `/` or NUL.
- `netns.go:CreateNamed` - creates a named netns and restores the caller's netns before returning; removes a leftover name of the same id first.
- `netns.go:RemoveNamed` - unmounts (MNT_DETACH, UMOUNT_NOFOLLOW) and unlinks `/run/netns/<name>`; ENOENT/EINVAL are not errors.
- `netns.go:Do` - runs a function with the OS thread switched into the target netns.
- `netns.go:Listen` - opens wildcard TCP listeners inside the netns; sockets keep their namespace and can be served from another.
- `dial.go:Dialer` - dial function for TCP/UDP IP literals created inside the netns; the OS thread is held only until socket creation via `net.Dialer.ControlContext` (`dial.go:with`, `dial.go:validateDialTarget`).
- `sysctl.go:SetSysctls`/`sysctl.go:Sysctl` - writes sysctls in the current netns, remounting `/proc/sys` read-write once on EROFS (`sysctl.go:procSys`).

## Invariants

- A failed netns restore panics rather than leaving one OS thread in the wrong namespace (`netns.go:CreateNamed`, `netns.go:Do`).
- `CreateNamed` removes a stale name before creating: the kernel's O_EXCL would otherwise wedge the name after one failed teardown (`netns.go:CreateNamed` doc).
- `Dialer` accepts only `tcp*`/`udp*` networks and IP-literal hosts; a socket re-creation attempt outside the namespace is an error (`dial.go:validateDialTarget`, `dial.go:Dialer` `ControlContext`).
- `SetSysctls` writes every key even if it already reads that way (writing one key can change another) and skips missing `net/ipv6/` keys; all writes under one `sysctl.go:procSysMu` lock hold (`sysctl.go:set` doc).
- Linux-only (`//go:build linux` on netns.go, dial.go, sysctl.go).

## Side Effects

- Creates/removes mounts and files under `/run/netns` (`netns.go:CreateNamed`, `netns.go:RemoveNamed`).
- Remounts `/proc/sys` read-write, then back to read-only, when a sysctl write hits EROFS (`sysctl.go:procSys.remount`).
- Opens sockets (listeners, dials) inside the target namespace (`netns.go:Listen`, `dial.go:Dialer`).

## Verification

- Run: `go test ./internal/netns/...`
