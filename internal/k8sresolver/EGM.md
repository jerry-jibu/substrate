# k8sresolver

## Overview

gRPC name resolver for the `k8s://` scheme, resolving Service names to
EndpointSlice addresses via client-go informers, so gRPC clients dial healthy
Kubernetes endpoints directly.

## Interface

- `Scheme = "k8s"` (`resolver.go:Scheme`).
- `Builder` / `NewBuilder(client kubernetes.Interface) *Builder` (`resolver.go:NewBuilder`): `resolver.Builder` over EndpointSlices; `Build` starts a namespaced informer and returns the live resolver.
- `ParseTarget(target resolver.Target) (ns, svc, port string, err error)` (`resolver.go:ParseTarget`): parses `k8s:///<namespace>/<service>[:port]` or `k8s:///<service>.<namespace>[:port]`.

## Invariants

- `Build` fails without a Kubernetes client (`resolver.go:Build`).
- Only endpoints that are ready or serving are returned; terminating-but-serving pods keep receiving traffic so connections are not dropped (`resolver.go:isHealthy`).
- Port resolution order: target port, else the slice's first port (named or not, whichever is set), else 443 (`resolver.go:updateState`).
- Addresses are sorted and deduplicated before each `UpdateState` (`resolver.go:updateState`).
- Watch errors and cache-sync failures are reported to the gRPC ClientConn (`resolver.go:Build`).
- `Close` cancels the informer context, shuts down the work queue, and removes the event handler (`resolver.go:Close`).

## Side Effects

- Runs a SharedInformerFactory watching EndpointSlices in the target namespace and a work-queue goroutine that pushes state updates to the ClientConn (`resolver.go:Build`, `resolver.go:run`).

## Verification

- Run: `go test ./internal/k8sresolver/...`
