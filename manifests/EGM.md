# manifests

Kubernetes YAML for deploying Agent Substrate. Everything installable is
grouped under `ate-install/`, which the installer (`cmd/ate-setup`) and the
Kind dev path apply into the `ate-system` namespace.

## Subdirectories & Boundaries

- [./ate-install/EGM.md](./ate-install/EGM.md) - the install manifest tree: one manifest per component, the default-deny NetworkPolicy, per-environment overlays (Kind, GKE sizes, agentgateway), and generated CRD/Role manifests.
- `egress-credential-injection/` - credential-injection manifest fragments; no contract (non-code YAML, pruned).

## System Verification

- This tree is applied to a cluster by the installer (`go run ./cmd/ate-setup deploy ate-system`); the e2e wrappers in [../hack/EGM.md](../hack/EGM.md) require an already-installed cluster. There is no standalone test command for the YAML itself.
