# ate-setup/internal/cmd

## Overview

Implements the `ate-setup` cobra command tree: root command wiring, persistent
flags, and the `deploy`, `delete`, `setup`, `publish`, `create`, `demo`, and
`benchmarks` subcommands. Each subcommand delegates the actual cluster work to
`steps.Env` methods and reads its settings from `config.Resolved`.

## Interface

- `Execute()` / `Root()` (`root.go`) - run the command tree, or hand it to tests in-process.
- `rootCmd` (`root.go`) - root command; `PersistentPreRunE` loads config, reports it, optionally provisions cluster credentials, and builds the shared `steps.Env`.
- `NoReportDefaultsEnv` (`root.go`) - `ATE_NO_REPORT_DEFAULTS` shortens the configuration report.
- `deployAteSystemCmd`, `deployAteletCmd`, `deployAPIServerCmd`, `deployControllerCmd`, `deployAtenetCmd`, `deployPodCertControllerCmd`, `deploySandboxConfigCmd`, `deployPostgresCmd` (`deploy.go`) - deploy subcommands.
- `deployOptions()`, `sandboxClassesOption()`, `validateSandboxClassesFlag()` (`deploy.go`) - read and validate `csi.setup` and `sandboxClasses` settings.
- `setupCSICmd` / `csiDriver()` (`setup.go`) - `setup csi [nfs|hostpath|both|none]`, default `nfs`.
- `publishWorkerImagesCmd`, `publishReleaseImagesCmd`, `publishReleaseBinariesCmd` (`publish.go`) - build and push worker/release images and kubectl-ate release binaries.
- `deleteAteSystemCmd`, `deleteAtenetCmd`, `deleteAllCmd` (`delete.go`) - delete subcommands; `delete all` runs every demo deleter first.
- `createCmd` (`create.go`) - recreates single secrets/configs (jwt-authority-pool, actor-id-ca-pool, actor-id-ca-certs, egress-mitm-ca-pool, podcertificate-controller-cas, api-server-env-vars, api-authentication-config, api-config) via `steps.Env` methods.
- `deployDemoCmd` / `deleteDemoCmd` (`demo.go`) - one subcommand per `demos.All()` entry, named without the `demo-` prefix.
- `deployBenchmarksCmd` / `deleteBenchmarksCmd` / `benchmarkOptions()` (`benchmarks.go`) - deploy/delete the locust load test stack; validates `benchmark.workerCount` and `benchmark.sandboxClass`.

## Invariants

- Every `config.Registry` entry is bound to exactly one flag (`root.go` `init` calls `config.BindFlags`), and `Execute` fails before running if `config.Validate` reports a clash.
- The nearest `record` annotation on the command chain decides recording; the `deploy` subtree opts in once and demos/benchmarks opt back out (`root.go` `recordAnnotation`, `recordsRun`).
- The nearest `cluster` annotation decides whether a kube client is required; `publish` is annotated `cluster: no` and runs without a kubeconfig (`root.go` `clusterAnnotation`, `needsCluster`).
- Recording is best effort: an install that succeeded is never reported as failed because its record could not be written (`root.go` `recordRun`).
- Setting validation happens where the supplying channel is known, so errors name the flag/env/file that carried the bad value (`deploy.go` `deployOptions`, `benchmarks.go` `benchmarkOptions`).

## Side Effects

- Builds a Kubernetes client and may populate the kubeconfig or fetch GKE credentials in `PersistentPreRunE` (`root.go`, via `cfg.EnsureClusterCredentials` and `steps.NewEnv`).
- Mutates cluster state through `steps.Env` methods such as `DeployAteSystem`, `DeleteAll`, `SetupCSI`, `PublishReleaseImages`, and the `Create*` secret methods (`deploy.go`, `delete.go`, `setup.go`, `publish.go`, `create.go`).
- Writes run records to the record directory via `config.RecordSuccess` / `config.RecordFailure` (`root.go` `recordRun`); omits secret settings from failure records.
- Prints errors and exits with status 1 on failure (`root.go` `Execute`).

## Verification

- Run: `go test ./cmd/ate-setup/internal/cmd/...`
