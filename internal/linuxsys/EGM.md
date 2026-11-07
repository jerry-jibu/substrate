# linuxsys

## Overview

Adapts Linux-specific syscalls and socket options for the rest of Substrate.
The current surface reads the original destination of a transparently
redirected TCP connection. Files are Linux-only (`//go:build linux`).

## Interface

- `TCPOriginalDestination(conn net.Conn) (string, error)` (`original_dst_linux.go:TCPOriginalDestination`): reads the IPv4 or IPv6 destination preserved by a Linux REDIRECT rule, returned as `host:port`.

## Invariants

- Requires a `*net.TCPConn`; other connection types fail (`original_dst_linux.go:TCPOriginalDestination`).
- The IPv6 socket option (`ip6tSOOriginalDst = 80`, `original_dst_linux.go:ip6tSOOriginalDst`) is used only when the local address is not IPv4-mapped, so an AF_INET socket reports the real IPv4 error instead of EOPNOTSUPP (`original_dst_linux.go:TCPOriginalDestination`).
- A zero original port is an error (`original_dst_linux.go:formatOriginalDestination`).

## Side Effects

- Performs a `getsockopt(SO_ORIGINAL_DST / IP6T_SO_ORIGINAL_DST)` syscall on the connection's fd via `SyscallConn.Control` (`original_dst_linux.go:getOriginalDestination`).

## Verification

- Run: `go test ./internal/linuxsys/...`
