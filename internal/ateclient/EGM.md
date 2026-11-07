# ateclient

## Overview
Client construction for the ateapi gRPC Control service, used by `kubectl-ate` and other out-of-cluster clients. Dials directly to an endpoint or, when none is given, port-forwards to the ate-api-server pod and mints a ServiceAccount bearer token.

## Interface
- `builder.go:NewClient` - builds a `Client`; empty endpoint means automatic port-forward to the system namespace. Extra dial options are appended.
- `builder.go:Client` - wraps the gRPC connection; `Close` closes it and stops the port-forwarder; `WorkerService` returns a WorkerService client on the same connection (atelet only).
- `builder.go:LoadKubeConfig` - loads a Kubernetes rest config from a kubeconfig path and context.
- `builder.go:FallbackPrincipal` - the principal a client authenticates as with no token file.
- `builder.go:NamespaceEnv`, `builder.go:APIServiceEnv`, `builder.go:ClientServiceAccountEnv` - environment overrides (`ATE_NAMESPACE`, `ATE_API_SERVICE_NAME`, `ATE_CLIENT_SERVICE_ACCOUNT`).

## Invariants
- The minted token authenticates as `FallbackPrincipal` (the ate-client ServiceAccount), not as whoever runs the client; anyone RBAC-allowed to create that token gets the same identity (`builder.go:bearerTokenDialOption`).
- Dialing uses round-robin so traffic spreads across the headless Service's replicas instead of the first (`builder.go:roundRobinServiceConfig`).
- The server certificate is verified against the live servicedns trust bundle for the ateapi Service DNS name (`builder.go:serverTLSConfig`).
- Bearer credentials require transport security (`builder.go:bearerTokenCreds`).

## Side Effects
- Dials ateapi over gRPC, with optional tracing provider (`builder.go:NewClient`).
- Port-forwarding to the ate-api-server pod via the Kubernetes API when no endpoint is given (`builder.go:dialPortForward`).
- Mints a ServiceAccount token through a Kubernetes TokenRequest when no token file is given (`builder.go:bearerTokenDialOption`).
- Reads environment variables and the optional token file (`builder.go:systemNamespace`, `builder.go:readBearerToken`).

## Verification
- Run: `go test ./internal/ateclient/...`
