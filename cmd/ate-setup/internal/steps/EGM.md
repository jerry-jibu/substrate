# ate-setup/internal/steps

## Overview

Holds the install and teardown operations the shell installer implemented as
bash functions. Each exported function corresponds to one of those and keeps
its name in the `[step]` log line (`env.go` package doc).

## Interface

- `Env`, `NewEnv`, `NewLocalEnv` (`env.go`) - the shared execution context: resolved configuration plus cluster clients; `NewLocalEnv` builds one without a Kubernetes client for the publish commands.
- `Env` manifest plumbing: `ResolveAndApply`, `ResolveManifest`, `ResolveManifestBytes`, `ResolveAndApplyBytes`, `Kustomize` (`env.go`) - render, resolve `ko://` references, and apply.
- `Env` namespace helpers: `Namespace`, `EnsureAteSystemNamespace`, `RequireCanonicalNamespace`, `RequireKind`, `RequireControlPlane` (`env.go`, `preflight.go`).
- Timeouts and namespaces: `NamespaceTimeout`, `BootstrapTimeout`, `TrustBundleTimeout`, `DemoTimeout`, `NamespaceAteSystem`, `NamespacePodCert` (`env.go`).
- `DeployOptions`, `Validate`, `DeployAteSystem`, `DeployPodCertificateController`, `DeployAteAPIServer`, `DeployAteController`, `DeployAtelet`, `DeployAtenet`, `DeploySandboxConfig`, `EnsureCRDs`, `DeployCRDs` (`deploy.go`) - control-plane install and per-component redeploys.
- `DeleteAteSystem`, `DeleteTrustBundles`, `DeleteAtenet`, `Deleter`, `DeleteAll`, `WaitNamespacesDeleted` (`delete.go`) - teardown.
- Secret and config creation: `CreateJWTAuthorityPoolSecret`, `CreateActorIDCAPoolSecret`, `CreateEgressMITMCAPoolSecret`, `EnsureEgressMITMCAPoolSecret`, `CreatePodCertificateControllerCAs`, `CreateActorIDCACertsSecret`, `CreateAPIConfig`, `UpdateAPIConfig`, `CreateAPIAuthenticationConfig`, `UpdateAPIAuthenticationConfig` (`create.go`); `CreateAPIServerEnvVars`, `UpdateAPIServerEnvVars`, `EnsureEnvVarsSafeStandalone` (`apienv.go`).
- `DeployPostgres` (`postgres.go`) - the experimental single-replica PostgreSQL StatefulSet; Cloud SQL sidecar wiring in `cloudsql.go`.
- `EnsureAPIServerPrerequisites`, `EnsurePodCertificateCAs`, `WaitForPodCertificateTrustBundles` (`prereqs.go`).
- `PublishWorkerImages`, `PublishReleaseBinaries`, `PublishReleaseImages`, `PublishReleaseOptions` (`publish.go`).
- Substrate API helpers: `AteClient`, `EnsureAtespace`, `ActorTemplateFromManifest`, `CreateActorTemplate`, `CreateActor`, `CreateActorEgressPolicy`, `WaitActorTemplateGolden`, `DeleteSubstrateDemo`, `TemplateRef` (`substrate.go`, `actors.go`).
- `ValidateSandboxClasses` (`microvm.go`), `ValidateCSIDriver`, `CheckCSIDriver`, `SetupCSI` (`csi.go`), `BenchmarkOptions`, `DeployBenchmarks`, `DeleteBenchmarks` (`benchmarks.go`), `SystemOverlay` (`overlay.go`), `SubstrateVersion` (`version.go`).
- Unexported helpers: `deployIDPServer` / `renderIDPServer` (`idpserver.go`) and `injectPreviewArg` / `previewArg` (`preview.go`), which stamps `--preview` on every substrate container before image resolution.

## Invariants

- Existing CRDs and RBAC change only through `DeployCRDs`; a component redeploy cannot move them (`deploy.go` `EnsureCRDs`).
- An existing JWT authority pool is kept: regenerating it would rotate the root out from under actors (`create.go` `CreateJWTAuthorityPoolSecret`).
- Generated Secrets are created, not applied, because they hold private key material that must not be merged or overwritten (`create.go`).
- A ClusterTrustBundle is deleted only when both its name and its signer match, so teardown cannot delete a bundle another actor installed (`delete.go` `DeleteTrustBundles`).
- The image resolver is created lazily so steps that only apply static manifests work on a machine with no registry credentials (`env.go` `Env.resolver`).
- `injectPreviewArg` must run before image resolution, which rewrites the `ko://` references it keys on (`preview.go`).

## Side Effects

- Applies and deletes Kubernetes objects through `kube.Client`, runs ko and docker buildx via the resolver, and calls the ate API through `AteClient`.
- Generates CA pools, JWT signing keys, and trust bundles and writes them as Secrets (`create.go`, `prereqs.go`).
- Publishes images and binaries to registries and output directories (`publish.go`).

## Verification

- Run: `go test ./cmd/ate-setup/internal/steps/...`
