# portforward

## Overview

Tunnels to the pods behind a Service. The Kubernetes port-forward API is
pod-scoped, so forwarding to a Service resolves its selector to one ready
pod and its port mapping to a container port, then forwards to that pod —
the same thing kubectl does for `port-forward svc/<name>`.

## Interface

- `portforward.go:ServicePortForward` - forwards a random local port to a Service's `servicePort`, resolving named and defaulted targetPorts against the chosen pod (`portforward.go:resolveTargetPort`); returns the local port and a stop func.
- `portforward.go:ServicePodPortForward` - forwards straight to a containerPort on a ready backing pod, for ports the Service does not publish.
- `portforward.go:PodPortForward` - SPDY-over-websocket port forward to one pod (`portforward.NewSPDYOverWebsocketDialer`), binding `0:<targetPort>` and returning the actual local port.
- `portforward.go:firstReadyPodForService` - picks the first ready pod matching the Service selector (`portforward.go:IsPodReady`).
- Subdirectories: none.

## Invariants

- The tunnel is pinned to the picked pod; it does not fail over if that pod later dies (`portforward.go:ServicePortForward` doc).
- A Service without a selector, a Service port with no match, or a named targetPort missing on the pod is an error, not a guess (`portforward.go:firstReadyPodForService`, `portforward.go:resolveTargetPort`).
- Readiness means Phase Running, no deletion timestamp, and PodReady condition True (`portforward.go:IsPodReady`).

## Side Effects

- Calls the Kubernetes API (get Service, list Pods) and opens a websocket SPDY tunnel to the pod's portforward subresource; the forwarded listener stays open until the returned stop func closes it (`portforward.go:PodPortForward`).

## Verification

- Run: `go test ./internal/portforward/...`
