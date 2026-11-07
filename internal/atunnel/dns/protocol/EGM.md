# protocol

## Overview

DNS wire-format helpers shared by the atunnel DNS filter: packet sanity checks,
transaction-ID rewriting, synthesized error/NXDOMAIN responses, and RFC 1035
TCP length-prefixed framing.

## Interface

- `TypeIXFR dnsmessage.Type = 251` (`protocol.go:TypeIXFR`): incremental zone transfer QTYPE (RFC 1995).
- `RequestSanityCheck(raw []byte) bool` (`protocol.go:RequestSanityCheck`): true for packets of at least header length with the QR bit clear (query).
- `ResponseSanityCheck(raw []byte) bool` (`protocol.go:ResponseSanityCheck`): true for response packets with QDCount == 1.
- `SetTxnID(raw []byte, id uint16)` (`protocol.go:SetTxnID`): overwrites the transaction ID at bytes 0..1.
- `QDCount(raw []byte) uint16` (`protocol.go:QDCount`): reads QDCOUNT from bytes 4..5.
- `NXDomainPacket(hdr dnsmessage.Header, q dnsmessage.Question) []byte` (`protocol.go:NXDomainPacket`): serialized NXDOMAIN response to `q`.
- `ErrorPacket(hdr dnsmessage.Header, rcode dnsmessage.RCode, q *dnsmessage.Question) []byte` (`protocol.go:ErrorPacket`): serialized error response for `rcode`.
- `ReadTCPFrame(r io.Reader) ([]byte, error)` (`tcp.go:ReadTCPFrame`): reads one 2-byte big-endian length-prefixed DNS message.
- `WriteTCPFrame(w io.Writer, msg []byte) error` (`tcp.go:WriteTCPFrame`): writes `msg` with its 2-byte big-endian length prefix.

## Invariants

- Sanity checks reject any packet shorter than the 12-byte DNS header (`protocol.go:RequestSanityCheck`, `protocol.go:ResponseSanityCheck`).
- `NXDomainPacket` carries a root-zone SOA with TTL 0 and MinTTL 0 so resolvers do not cache the negative answer (RFC 2308 section 5) (`protocol.go:NXDomainPacket`).
- `ErrorPacket` always returns a usable packet: on builder failure it synthesizes a minimal 12-byte header response (`protocol.go:ErrorPacket`).
- `WriteTCPFrame` rejects messages longer than 65535 bytes (`tcp.go:WriteTCPFrame`).

## Side Effects

- None. Pure functions over caller-owned byte slices and `io.Reader`/`io.Writer` arguments.

## Verification

- Run: `go test ./internal/atunnel/dns/protocol/...`
