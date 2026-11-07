# router

## Overview
The atenet-router: an Envoy control plane and ext_proc server for actor traffic. `router.go:RouterServer` coordinates the runtime threads; `xds.go:XdsServer` serves Envoy xDS configuration; the ext_proc handlers under `ingress` and `egress` implement actor ingress (resume-on-demand, request parking) and egress policy enforcement. `cmd.go:NewRouterCmd` is the cobra entry point.

## Interface
- `cmd.go:NewRouterCmd() *cobra.Command` registers the `atenet router` command.
- `router.go:RouterServer`, built with `router.go:NewRouterServer(cfg routerConfig)`; holds the ext_proc mux, the ingress handler, and the health checker.
- `xds.go:XdsServer`, built with `xds.go:NewXdsServer(xdsPort int)`, generates and serves the Envoy listeners, clusters, and routes; `dataplane.go` wires the data plane pieces.
- `config.go:Mode` selects ingress-only, egress-only, or combined serving.
- Health and status: `health.go:RouterHealthReport` and `ComponentHealth`, `status.go:DashboardContext` and `FormattedQuery` for the status page.
- Static-resource validation: `envoyadmin_manifest_test.go`, `egressmanifest_test.go`, `egresspolicymanifest_test.go`, `ingressauthmanifest_test.go` validate the Envoy static YAML manifests that ate-setup installs under `manifests/ate-install/`; `drain.go` and `envoydrain.go` implement connection drain.
- `loopback.go` serves the loopback (pod-internal) listener.
- Children: `./egress/EGM.md`, `./ingress/EGM.md`, `./extproc/EGM.md`.

## Invariants
- Only ingress mode needs Kubernetes access; an egress-only instance deliberately runs without cluster access and its client is never built (`router.go:NewRouterServer` guard).
- Dispatch is by filter chain, not HTTP method, because ingress and egress apply different policy families (`extproc/dispatch.go:Direction` doc).
- Parentless data plane requests default to 1% root trace sampling (`router.go:dataPlaneTraceRatio`).

## Side Effects
- Serves Envoy xDS over gRPC on the configured xDS port.
- Serves the ext_proc gRPC service and the status/health HTTP endpoints.
- Reads EndpointSlices and Secrets from the Kubernetes API in ingress mode.
- Calls the ateapi Control API to resume actors and fetch policies.

## Verification
- Run: `go test ./cmd/atenet/internal/router/...`
