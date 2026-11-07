# src

## Overview

Rust sources of the `substrate-envoy-egress-policy` dynamic module
(cdylib `envoy_substrate_egress_policy`): an HTTP filter that stores egress
policies parsed from Envoy dynamic metadata, a worker-thread-local policy
store, and a listener filter that picks the filter chain from the policy.

## Interface

- `lib.rs:init` - module load hook; `lib.rs:declare_all_init_functions!` registers `policy_store_filter::new_http_filter_config_fn` (http) and `listener_filter::new_listener_filter_config_fn` (listener).
- `egress_policy.rs:EgressPolicy` - SNI rules parsed from JSON; `egress_policy.rs:SniRule` is `{pattern, mode}` where mode is `mitm` or `passthrough`.
- `egress_policy.rs:ATE_POLICY_EGRESS_NAMESPACE` (`dev.ate.policy.egress`) and `egress_policy.rs:ATE_POLICY_JSON_KEY` (`policy_json`) - dynamic metadata location of the policy JSON.
- `egress_policy.rs:EgressPolicy::new_from_metadata` - parses the policy, blanks the metadata JSON, and hashes the policy with Rapidhash into `EgressPolicy.hash`.
- `egress_policy_store.rs:GlobalEgressPolicyStore`/`GLOBAL_EGRESS_POLICY_STORE`/`get_local_store` - process-wide, thread-local store keyed by policy hash (`egress_policy_store.rs:EgressPolicyStore.lookup`/`store`/`remove`).
- `policy_store_filter.rs:EgressPolicyStoreFilter` - on request headers, parses the policy, stores or shares it, writes the hash to filter state `policy_store_filter.rs:ATE_POLICY_EGRESS_HASH_NOT_SHARED`, and removes the entry in `on_stream_complete` when it holds the last reference.
- `policy_store_filter.rs:EGRESS_POLICY_STORE_SIZE`/`EGRESS_POLICY_PARSE_ERRORS`/`EGRESS_POLICY_SHARED` - stats `ate_egress.policy_store_size`, `ate_egress.parse_errors`, `ate_egress.policy_shared`.
- `listener_filter.rs:EgressPolicyFilter` - runs after tls_inspector; `listener_filter.rs:transport_verdict` and `listener_filter.rs:tls_verdict` decide the chain written to filter state `listener_filter.rs:ATE_EGRESS_FILTER_CHAIN`.
- `listener_filter.rs:pattern_matches` - wildcard matching (`*`, or `*.suffix` = exactly one non-empty leftmost label); must agree with `HostnamePattern.Matches` in `internal/egresspolicy`.
- Subdirectories: none.

## Invariants

- Rules are most-specific-first; the first match wins and is final even with an unknown mode (`egress_policy.rs:EgressPolicy.rules`, `listener_filter.rs:tls_verdict`).
- TLS without a policy or SNI is denied; unknown transports are denied; non-TLS is cleartext (`listener_filter.rs:transport_verdict`, `listener_filter.rs:tls_verdict`).
- SNI is ASCII-lowercased with one trailing dot stripped; normalization is the filter's job, not `pattern_matches`' (`listener_filter.rs:normalize_sni`).
- A hash collision (same hash, different policy) fails closed with HTTP 400 `hash_collision` (`policy_store_filter.rs:EgressPolicyStoreFilter.on_request_headers`).
- The policy store is thread-local and never crosses threads; entries are never dropped while a filter on the thread holds them (`egress_policy_store.rs:unsafe impl Send`, `policy_store_filter.rs:on_stream_complete`).
- The SDK version in `../Cargo.toml` must not be newer than the Envoy in `../../Dockerfile`, or the module will not load (`../Cargo.toml` workspace.dependencies comment).

## Side Effects

- Mutates Envoy dynamic metadata (blanks the policy JSON after parse), filter state, and the `ate_egress.*` stats (`egress_policy.rs:EgressPolicy::new_from_metadata`, `policy_store_filter.rs:EgressPolicyStoreFilter`).
- Writes the chain verdict to filter state consumed by Envoy filter chain selection (`listener_filter.rs:EgressPolicyFilter.on_accept`).

## Verification

- Run: `cd cmd/dataplane/envoy/dynamic-modules && cargo test -p substrate-envoy-egress-policy`
