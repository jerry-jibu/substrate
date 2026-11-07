# infra

## Overview

Infrastructure API definitions: the Kubernetes CRD types under `k8s/` that
describe the substrate's own infrastructure (worker pools, sandbox runtime
configs, CSI driver configs).

## Subdirectories & Boundaries

- k8s: [./k8s/EGM.md](./k8s/EGM.md) - Kubernetes API groups; `k8s/ate/v1`
  holds the `ate.dev/v1` CRD types.
