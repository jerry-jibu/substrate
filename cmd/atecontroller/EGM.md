# atecontroller

## Overview
The ate-controller binary. `main.go` builds a controller-runtime manager that runs the CRD reconcilers in `internal/controllers` and the worker syncer in `internal/workersync`, and dials ate-api-server with a pod-identity client certificate. `metrics.go` pads empty exponential histograms for OTLP export.

## Interface
- `main.go` flags define the deployment surface: `--ateapi-conn-spec`, `--ateapi-ca-file`, `--ateapi-server-name`, `--ateapi-client-cert` (required), `--worker-sync-concurrency`, `--dns-egress-policy`, `--atelet-service-account`, `--router-service-account`, `--preview`, `--log-level`, `--version`, plus the `--otel-*` flags propagated onto worker Pods.
- `main.go:init` registers the `atev1` CRD scheme (`atev1.AddToScheme`).
- `main.go` configures the manager's metrics listener (`main.go:metricsPullAddr`, `main.go:metricsBindAddr`); `metricsBindAddr` returns `"0"` (listener disabled) when OTLP push means no pull listener is needed.

## Invariants
- `--ateapi-client-cert` is required; the controller authenticates to ate-api-server with it (`main.go` flag doc).
- `--dns-egress-policy` must be `egresspolicy.DNSEgressPolicyEnforceInAtunnel` (`main.go` flag doc).
- The atelet and router service account flags must match the SPIFFE IDs workers' atunnel expects (`main.go` flag docs).

## Side Effects
- Runs controller-runtime reconcilers against the cluster and serves manager metrics on :8080 unless disabled under OTLP push (`main.go:metricsBindAddr`).
- Deletes terminal and orphaned worker Pods through `internal/workersync`.

## Verification
- Run: `go test ./cmd/atecontroller/...`
