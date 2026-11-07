# dynamic-modules

## Overview

Cargo workspace for Envoy dynamic modules used by the dataplane image.
Contains one member crate today, `egress-policy`.

## Interface

- `Cargo.toml:[workspace]` - resolver 3, members `egress-policy`; workspace version 0.1.0, edition 2024.
- `Cargo.toml:[workspace.dependencies]` - pins `envoy-proxy-dynamic-modules-rust-sdk` to commit `b579d07d3ad7ee11d32b105e91a5a39ad24718d7` (Envoy v1.39.1); the SDK must not be newer than the Envoy in `../Dockerfile`.
- Subdirectories: [./egress-policy/EGM.md](./egress-policy/EGM.md) - the egress-policy listener/HTTP filter module (sources under `egress-policy/src/`).

## Invariants

- SDK and Envoy versions in the dataplane image must stay in lockstep, per the comment on the pinned dependency (`Cargo.toml`).

## Side Effects

- ["None"]

## Verification

- Run: `cd cmd/dataplane/envoy/dynamic-modules && cargo test`
