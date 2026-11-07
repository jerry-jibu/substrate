# credential-provider

## Overview
Parent directory for credential-provider plugin binaries implementing the `api/credprovider` plugin API. Currently holds the Kubernetes Secrets backend.

## Subdirectories & Boundaries
- kubernetes-secrets: [./kubernetes-secrets/EGM.md](./kubernetes-secrets/EGM.md) - the `k8s.io` provider resolving `ate-secret://` URIs to Secret values for the egress credential injector.
