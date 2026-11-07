# credential-provider/kubernetes-secrets

## Overview
The `k8s.io` credential-provider plugin: a gRPC service resolving `ate-secret://k8s.io/default/<namespace>/<secret>/<key>` URIs to Kubernetes Secret values. It is the only component in the egress credential-injection path with Kubernetes access; the egress gateway and its injector never read Secrets directly.

## Interface
- `main.go:main` / `main.go:run` - gRPC server wiring with mTLS, metrics, health, and a `/statusz` page.
- Flags (`main.go` var block): `--listen-address` (:50051), `--metrics-address`, `--health-address`, `--status-address`, `--server-cred-bundle` (required), `--client-ca-file` (required), `--injector-identity` (SPIFFE ID of the allowed injector), `--namespace-policy-file` (required), `--log-level`, `--drain-grace`, `--kube-api-qps`, `--kube-api-burst`, `--preview`.
- `kubeprovider.go:Server` - implements `credprovider.CredentialProviderServer` over the Kubernetes API.
- `kubeprovider.go:ParseURI` / `kubeprovider.go:SecretRef` - parse and carry the credential URI; consts `kubeprovider.go:ProviderName` ("k8s.io") and `LocalLocator` ("default").
- `nsauthz.go:NamespaceAuthorizer` / `nsauthz.go:LoadNamespaceAuthorizer` - atespace-to-namespace grant policy from YAML.
- `statusz.go:newStatuszHandler` - `/statusz` dashboard, JSON under `?format=json`.

## Invariants
- `nsauthz.go:NamespaceAuthorizer` is default-deny: an atespace absent from the policy resolves nothing. Grants for one atespace/namespace are OR-ed; label criteria within a grant are AND-ed.
- `nsauthz.go:LoadNamespaceAuthorizer` parses strictly, so a typo or unsupported field fails startup rather than the first request.
- `kubeprovider.go:ParseURI` rejects any scheme other than `ate-secret`, any provider other than `k8s.io`, query/fragment components, percent-encoded paths, empty segments, non-`default` locators, and paths that are not exactly `<namespace>/<secret>/<key>`.
- The injector is the only caller allowed to fetch secrets; its SPIFFE identity is pinned by `--injector-identity` (`main.go` flag comment).
- `/statusz` shows names and configuration only, never secret values (`statusz.go` package doc).

## Side Effects
- Reads Secrets from the Kubernetes API server (rate-limited by `--kube-api-qps`/`--kube-api-burst`).
- Serves gRPC with mTLS, Prometheus `/metrics`, `/readyz`/`/healthz` (started only after the gRPC listener opens), and `/statusz` on loopback.
- Drains in-flight RPCs for `--drain-grace` on SIGINT/SIGTERM.

## Verification
- Run: `go test ./cmd/credential-provider/kubernetes-secrets/...`
