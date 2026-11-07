# controllers

## Overview
Controller-runtime reconcilers for ate-controller: they reconcile WorkerPool CRDs into worker Deployments, per-WorkerPool NetworkPolicies, SandboxConfig deletion protection, and the egress MITM trust config.

## Interface
- `workerpool_controller.go:WorkerPoolReconciler` reconciles `atev1.WorkerPool`; entry points `workerpool_controller.go:WorkerPoolReconciler.Reconcile`, `InitMetrics`, `SetupWithManager`. `SetupWithManager` owns `appsv1.Deployment` (`workerpool_controller.go:442`).
- `workerpool_apply.go` builds the worker Deployment with a RollingUpdate strategy (`workerpool_apply.go:280`).
- `networkpolicy_controller.go:NetworkPolicyReconciler.Reconcile` manages the per-WorkerPool NetworkPolicy; `SetupWithManager` at `networkpolicy_controller.go:177`.
- `sandboxconfig_protection_controller.go:SandboxConfigProtectionReconciler.Reconcile` guards SandboxConfig deletion and defaults; `SetupWithManager` at `sandboxconfig_protection_controller.go:133`.
- `egressmitmtrust_controller.go:EgressMITMTrustReconciler.Reconcile` maintains the egress MITM CA trust config; `egressmitmtrust_controller.go:EgressMITMCAPoolRef` names the CA pool object. `SetupWithManager` at `egressmitmtrust_controller.go:197`.
- `gen.go` carries the kubebuilder RBAC markers controller-gen cannot scan (worker pod list/watch/delete, ReplicaSet get, EndpointSlices) and the `go:generate` directive for RBAC manifests.

## Invariants
- WorkerPool state is written through `Status().Update` (`workerpool_controller.go:379`).
- Actors must run on both gVisor and microVM sandboxes; the reconciler maps WorkerPool sandbox config to SandboxConfigs in `workerpool_sandboxconfig.go`.

## Side Effects
- Creates and updates Deployments, NetworkPolicies, and SandboxConfig protection objects in the API server.
- Registers OTel metrics via `workerpool_controller.go:InitMetrics`.

## Verification
- Run: `go test ./cmd/atecontroller/internal/controllers/...` (envtest-based).
