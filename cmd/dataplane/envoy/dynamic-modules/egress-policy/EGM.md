# egress-policy

## Overview

The `substrate-envoy-egress-policy` Envoy dynamic module crate, a cdylib
built by the `cmd/dataplane/envoy/dynamic-modules` Cargo workspace. Module
design and build details are in `README.md` in this directory.

## Interface

- `Cargo.toml:[package]` - crate `substrate-envoy-egress-policy`, lib name `envoy_substrate_egress_policy`, `crate-type = ["cdylib"]`.
- `Cargo.toml:[dependencies]` - `envoy-proxy-dynamic-modules-rust-sdk` (pinned by the workspace to an Envoy v1.39.1 commit), `rapidhash`, `serde`, `serde_json`, `thread_local`.
- Subdirectories: [./src/EGM.md](./src/EGM.md) - the module's filters, policy types, and store.

## Invariants

- SDK version is pinned by the workspace `Cargo.toml` and must not be newer than the Envoy in the dataplane image (`../Cargo.toml`).

## Side Effects

- ["None"]

## Verification

- Run: `cd cmd/dataplane/envoy/dynamic-modules && cargo test -p substrate-envoy-egress-policy`
