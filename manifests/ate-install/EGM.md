# ate-install

## Overview

Kubernetes YAML for deploying Agent Substrate into the `ate-system`
namespace: one manifest per component, plus the default-deny NetworkPolicy
baseline and the SandboxConfig definitions the sandbox runtimes read.

## Interface

- `ate-system-namespace.yaml` - the `ate-system` namespace.
- `ate-api-server.yaml`, `ate-controller.yaml`, `atelet.yaml`, `ate-idp-server.yaml` - Deployments/DaemonSets of the control plane and node agent.
- `pod-certificate-controller.yaml` - the pod certificate controller deployment.
- `atenet-router.yaml`, `atenet-egress.yaml` - the gateway deployments; both carry an `${ENVOY_DATAPLANE_IMAGE}` placeholder that install substitutes with the built `envoy-dataplane` digest (`cmd/dataplane/envoy/README.md`).
- `atenet-router-monitoring.yaml` - monitoring for the router.
- `ate-system-networkpolicies.yaml` - default-deny ingress across `ate-system` (the network-isolation baseline named in AGENTS.md).
- `ate-otel-config.yaml` - OTel collector configuration.
- `sandboxconfig-gvisor.yaml`, `sandboxconfig-microvm.yaml.tmpl`, `sandboxconfig-validation.yaml` - SandboxConfig resources for the gVisor and microVM classes (the template is rendered at install) and the validation config.
- Subdirectories: [./generated/EGM.md](./generated/EGM.md) - codegen output (CRDs, RBAC).

## Invariants

- Default-deny ingress across `ate-system` is the baseline; every allowed flow must be an explicit rule in `ate-system-networkpolicies.yaml` (AGENTS.md Security Considerations).
- Image references are pinned by digest (AGENTS.md Code rules).

## Side Effects

- Applying these files creates and updates cluster objects in `ate-system`; the manifests are the installer's source of truth (`cmd/ate-setup`).

## Verification

- No test runner: structure and references are verified by `hack/verify-all.sh` and the installer itself.
