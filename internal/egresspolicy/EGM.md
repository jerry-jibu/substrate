# egresspolicy

## Overview
Compiles and evaluates `EgressPolicy` objects for the atenet egress gateway: hostname and SNI pattern matching, port rules, and the `--dns-egress-policy` flag handling for DNS enforcement in the worker DNS relay.

## Interface
- `egresspolicy.go:Policy`, `egresspolicy.go:Compile` - parses every pattern and port once, ready for repeated evaluation; returns the policy plus per-entry errors.
- `egresspolicy.go:EvaluateRequest` - evaluates a `Destination` against the policy; `decrypted` selects https rules, otherwise http rules.
- `egresspolicy.go:RuleCount` - number of rules, dropped entries included.
- `egresspolicy.go:SNIRules`, `egresspolicy.go:SNIRule`, `egresspolicy.go:SNIMode` - SNI rules for a dialed port, sorted by the API tie-breaking rules.
- `egresspolicy.go:Destination`, `egresspolicy.go:Decision` - evaluation input and outcome.
- `egresspolicy.go:HostnamePattern`, `egresspolicy.go:ParseHostnamePattern` - parsed host/SNI patterns.
- `egresspolicy.go:NormalizeAuthority`, `egresspolicy.go:NormalizeHostname` - canonicalize authority/host values.
- `egresspolicy.go:DNSEgressPolicyEnforceInAtunnel`, `egresspolicy.go:ValidateDNSEgressPolicy` - the `--dns-egress-policy` mode constant and its validator.

## Invariants
- A dropped compile entry fails closed because it only narrows an allow rule; the `Policy` is always usable (`egresspolicy.go:Compile`).
- A policy with no rules can authorize nothing (`egresspolicy.go:RuleCount`).
- Patterns are lowercase DNS-1123 subdomains, optionally `*.`-prefixed to match exactly one non-empty leftmost label, or `*` alone; IP literals, ports, URLs, trailing dots, and other `*` placements are rejected (`egresspolicy.go:ParseHostnamePattern`).
- Anything `NormalizeAuthority` cannot parse is an error, and the caller should deny (`egresspolicy.go:NormalizeAuthority`).
- A `tls_passthrough` rule never decides a request (`egresspolicy.go:EvaluateRequest`).
- `SNIMode` values must match `cmd/dataplane/envoy/dynamic-modules/egress-policy` (`egresspolicy.go:SNIMode`).

## Side Effects
- None. Pure policy compilation and evaluation.

## Verification
- Run: `go test ./internal/egresspolicy/...`
