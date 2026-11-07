# ate

## Overview

API group directory for the `ate.dev` Kubernetes custom resources. The `v1`
directory holds the Go types that generate the `WorkerPool`, `SandboxConfig`,
and `CSIDriverConfig` CRDs.

## Subdirectories & Boundaries

- v1: [./v1/EGM.md](./v1/EGM.md) - `WorkerPool`, `SandboxConfig`, and
  `CSIDriverConfig` types for group `ate.dev/v1`, plus the codegen entry
  point.
