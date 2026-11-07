# generated

## Overview

Codegen output installed with the rest of `ate-install`: the CRD manifests
for the `ate.dev` group and the controller's ClusterRole. Regenerate via
the codegen targets (`api/infra/k8s/ate/v1/gen.go` per the api tree
contract) rather than hand-editing.

## Interface

- `ate.dev_workerpools.yaml` - the `WorkerPool` CRD.
- `ate.dev_sandboxconfigs.yaml` - the `SandboxConfig` CRD.
- `ate.dev_csidriverconfigs.yaml` - the `CSIDriverConfig` CRD.
- `role.yaml` - ClusterRole `ate-controller` (pods, serviceaccounts, deployments, and other rules the controller reconciles).
- `crds.go` - Go package `generated` embedding the CRD manifests via `crds.go:crdFiles` (`embed.FS`); `crds.go:CRDs` yields each manifest's raw text for tools such as apitool.

## Invariants

- Everything here is generated; edits are made in the API types and regenerated (`hack/verify/codegen.sh` checks drift).

## Side Effects

- Installer applies the CRDs and ClusterRole to the cluster.

## Verification

- No test runner; `hack/verify/codegen.sh` (via `hack/verify-all.sh`) verifies freshness.
