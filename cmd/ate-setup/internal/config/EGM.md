# ate-setup/internal/config

## Overview

Resolves the environment ate-setup installs into. It layers the developer's
`.ate-dev-env.sh` (sourced through bash), the ambient process environment, an
optional configuration document, and command line flags into a single typed
`Config`, and records what a run used so it can be reproduced with `--config`
(`config.go` package doc).

## Interface

- `Registry`, `RegisterCommand`, `All`, `Validate`, `Lookup`, `Keys`, `BindFlags`, `BindCommandFlags` (`setting.go`) - the settings catalogue: one `Setting` declares key, env var, flag, kind, default, and owning command.
- `Setting`, `ValueKind` (`setting.go`) - declaration types.
- `LoadFlags`, `LoadOptions`, `ConfigPathEnv` (`load.go`) - resolve the effective configuration from a bound flag set; `ATE_CONFIG` names the document when `--config` is not given.
- `Origin`, `Value`, `Resolved`, `ResolveOptions`, `Resolve` and accessors `String`/`Bool`/`Int`/`Duration`/`Supplied`/`Require`/`Origins` (`resolve.go`) - layered resolution with per-key origin.
- `Config` and helpers `Size10`, `PostgresSchemaName`, `WaitTimeout`, `Manifest`, `Path`, `ScriptEnv`, `KoEnv`, `CredentialProviders`, `HasKubernetesProvider`, `RequireTransferServiceAccount`, `SetResolved`, `Resolved` (`config.go`) - the typed install environment; enum constants `RouterEnvoy`/`RouterAgentgateway`, `SandboxClassGvisor`/`SandboxClassMicrovm`, `ClusterSizeSize0`/`ClusterSizeSize10`, `DefaultRolloutTimeout`, `Size10PostgresPoolParams`.
- `CloudSQLConfig` (`config.go`) - the operator's `ATE_API_POSTGRES_CLOUDSQL_*` intent; `CredentialProvider` (`config.go`) - parsed `--credential-provider`.
- `ParseFile`, `File`, `Get` (`file.go`) - parse a configuration document; unknown keys are rejected.
- `Document`, `DocumentMetadata`, `Outcome`, `NewDocument`, `Marshal` (`document.go`) - render a resolved configuration as a reproducible document.
- `RecordDir`, `RecordSuccess`, `RecordFailure`, `ClusterKey` (`record.go`) - write run records.
- `Report`, `ReportOptions` (`report.go`) - print the settings a run is about to apply and their origins.
- `EnsureClusterCredentials` (`credentials.go`), `CheckKindCluster` (`kindcheck.go`), `RepoRoot` (`shellenv.go`) - GKE credential fetch, kind prerequisite check, module root discovery.
- `RequiredError`, `InvalidError`, `ConflictError` (`errors.go`) - errors that name the channel that supplied a bad value.

## Invariants

- A setting reaches flag, env var, and document only through its one `Setting` declaration; `Validate` (`setting.go`) reports duplicate names and missing channels across the merged set.
- Channel precedence is fixed, lowest to highest: `.ate-dev-env.sh`, process environment, configuration document, flags (`load.go` `LoadFlags`, `resolve.go` `Resolve`); a channel that does not carry a setting leaves the lower layer in place.
- Unknown keys in a configuration document are rejected rather than ignored (`file.go` `ParseFile`); a key given twice in one document is an error (`file.go`).
- Documents omit settings still on their default and omit secret settings whatever supplied them, reporting the omitted secret keys (`document.go` `NewDocument`).
- A failed run's record must not change what a later run resolves: `RecordFailure` writes under a name nothing reads (`record.go`).
- Secret settings never get flag shorthand; only bools get a typed flag, ints and durations parse via `Setting.parse` (`setting.go`).

## Side Effects

- Sources `.ate-dev-env.sh` through bash, capturing its stdout separately from the records stream (`shellenv.go`, `load.go`); skipped for Kind installs and `--no-dev-env`.
- `EnsureClusterCredentials` runs `gcloud` to fetch GKE credentials and updates the kubeconfig (`credentials.go`).
- `RecordSuccess`/`RecordFailure` write files into the record directory, overwriting the per-context success record (`record.go`).
- `Report` writes the configuration report to the supplied writer (`report.go`).
- `ScriptEnv` and `KoEnv` (`config.go`) export the variable sets the legacy shell scripts and ko builds read.

## Verification

- Run: `go test ./cmd/ate-setup/internal/config/...`
