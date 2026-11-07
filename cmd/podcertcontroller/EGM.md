# podcertcontroller

## Overview
Pod certificate controller implementing three signers backed by local CA pools: `servicedns.podcert.ate.dev/identity` (service DNS names), `podidentity.podcert.ate.dev/identity` (KSA-equivalent identities), and `postgres.podcert.ate.dev/identity` (PostgreSQL login certificates for ate-api-server).

## Interface
- `main.go:main` - CLI entry point; builds one `signercontroller.Controller` per signer and runs them with `--workers-per-signer` goroutines each.
- Flags (`main.go` var block): `--kubeconfig` / `--in-cluster`, `--sharding-pod-namespace` / `--sharding-pod-name` / `--sharding-pod-uid` / `--sharding-application-name`, `--service-dns-ca-pool`, `--pod-identity-ca-pool`, `--postgres-ca-pool`, `--postgres-client-namespace`, `--workers-per-signer`, `--kube-api-qps`, `--kube-api-burst`, `--version`, `--preview`.
- [internal](./internal/EGM.md) - signer controller, PCR client, rendezvous sharding, and the three signers.

## Invariants
- `main.go:main` rejects negative `--kube-api-qps` / `--kube-api-burst`; zero keeps the client-go defaults.
- CA pool files are loaded through `localca.NewRefreshingPool` at startup; failure exits (`main.go:main`).
- The PCR informer runs once, shared by all three signer controllers (`main.go:main` runs `pcrClient.Informer()` separately from each controller's `Run`).

## Side Effects
- Talks to the Kubernetes API server (PCRs, ClusterTrustBundles, Services, Leases).
- Rereads the three CA pool files from disk via refreshing pools (a TODO notes they do not reload on change yet).
- Exits on SIGINT/SIGTERM (`main.go:main`).

## Verification
- Run: `go test ./cmd/podcertcontroller/...`
