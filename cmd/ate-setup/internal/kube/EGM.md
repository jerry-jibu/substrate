# ate-setup/internal/kube

## Overview

Provides the cluster operations ate-setup needs, replacing the `kubectl`,
`kubectl wait`, and `kubectl exec` invocations of the shell installer
(`client.go` package doc).

## Interface

- `Client`, `New`, `NewFromClients` (`client.go`) - typed, dynamic, and discovery clients plus the RESTMapper used to route unstructured objects; `FieldManager` is `"ate-setup"` in managedFields.
- `InvalidateDiscovery`, `ServerVersion` (`client.go`) - drop cached discovery so newly installed CRDs resolve; connectivity check.
- `Apply`, `ApplyOne`, `ApplyPath`, `ApplyBytes`, `ApplyTolerant` (`apply.go`) - server-side apply of objects, manifest paths, or in-memory manifests, with a tolerant variant that skips kinds the cluster does not serve.
- `Delete`, `DeleteOne`, `DeletePath`, `DeleteBytes`, `Exists` (`apply.go`) - deletes ignoring NotFound; `DeletePath` treats a missing path as success.
- `RolloutStatus`, `WaitNamespaceActive`, `WaitJobComplete`, `WaitCondition`, `WaitClusterTrustBundles`, `WaitDeleted`, `RolloutRestartDeployment`, `RolloutRestart` (`wait.go`) - waits replacing `kubectl rollout status` / `kubectl wait`.
- `EnsureNamespace`, `ApplySecret`, `ApplyConfigMap`, `GetConfigMap`, `MergePatchConfigMap`, `GetSecret`, `SecretExists`, `ConfigMapExists`, `GetDeployment`, `DeploymentExists`, `DeleteDeploymentAndWait`, `PatchDeployment`, `SetServiceAccountAnnotation`, `ServiceAccountAnnotation`, `OIDCIssuer` (`resources.go`) - typed resource helpers.
- `DecodeManifest`, `DecodeManifestBytes`, `LoadPath`, `ReadPath`, `Describe` (`manifest.go`) - multi-document YAML decoding and directory loading that mirrors `kubectl apply -f <dir>`.
- `Exec` (`exec.go`) - run a command in a container with attached streams.
- `CAPoolRootPEM` (`capool.go`) - extract a CA pool Secret's root certificates PEM-encoded.

## Invariants

- Writes are server-side apply under the fixed field manager `ate-setup` (`client.go` `FieldManager`, `apply.go` `Apply`).
- Deletes ignore NotFound; a kind that no longer resolves counts as already deleted during teardown (`apply.go` `DeleteOne`).
- Directory loading mirrors `kubectl apply -f <dir>` semantics deliberately (`manifest.go` `LoadPath`).
- `WaitClusterTrustBundles` shares one deadline across every bundle, not a budget per bundle (`wait.go`).
- Restarting an absent DaemonSet is not an error, because the CSI setup restarts atelet only if present (`wait.go` `RolloutRestart`).

## Side Effects

- Mutates cluster state through the Kubernetes API (apply, delete, patch, wait, exec).
- `Exec` runs commands inside cluster containers (`exec.go`).

## Verification

- Run: `go test ./cmd/ate-setup/internal/kube/...`
