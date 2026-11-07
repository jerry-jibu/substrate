# envoy

## Overview

Build definition for the Envoy dataplane container image (`envoy-dataplane`)
used by the `atenet-router` and `atenet-egress` gateways. Multi-stage
`Dockerfile`: a `rust:bookworm` builder cross-compiles the Rust dynamic
module, and the runtime stage packages the `.so` into a pinned
`envoyproxy/envoy:v1.39.5` image.

## Interface

- `Dockerfile` stage `builder` - builds `libenvoy_substrate_egress_policy.so` with `cargo build --release --locked`, cross-compiling per `TARGETARCH` without emulation.
- `Dockerfile` runtime stage - base `envoyproxy/envoy:v1.39.5` (digest-pinned); installs the module at `/usr/local/lib/libenvoy_substrate_egress_policy.so` and sets `ENVOY_DYNAMIC_MODULES_SEARCH_PATH=/usr/local/lib`.
- Build entry: `docker buildx build -t envoy-dataplane cmd/dataplane/envoy` (see `README.md` here); `ate-setup` builds and pushes it during install and substitutes the digest for `${ENVOY_DATAPLANE_IMAGE}` in `manifests/ate-install/atenet-router.yaml` and `manifests/ate-install/atenet-egress.yaml`.
- Subdirectories: [./dynamic-modules/EGM.md](./dynamic-modules/EGM.md) - Cargo workspace holding the egress-policy module.

## Invariants

- Envoy version in the `Dockerfile` runtime stage and the SDK revision in `./dynamic-modules/Cargo.toml` must stay in lockstep (`Dockerfile` stage-2 comment).
- Base images are digest-pinned, and the Envoy image pins the multi-platform index, not one platform's manifest (`Dockerfile:FROM` comments).

## Side Effects

- `docker buildx` builds and `ate-setup` pushes `envoy-dataplane` images to `$KO_DOCKER_REPO` during install (documented in `README.md`).

## Verification

- Run: `cd cmd/dataplane/envoy/dynamic-modules && cargo test` (module unit tests; the image itself has no test runner)
