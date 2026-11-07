# podcertcontroller/internal

## Overview
Internal packages of the `podcertcontroller` binary: the PCR API client, the generic signing controller, the rendezvous work-sharding hasher, and the three signer implementations.

## Subdirectories & Boundaries
- podcertificate: [./podcertificate/EGM.md](./podcertificate/EGM.md) - version-selecting PodCertificateRequest client (v1 preferred, v1beta1 fallback).
- signercontroller: [./signercontroller/EGM.md](./signercontroller/EGM.md) - queue-driven controller that dispatches PCRs to a `SignerImpl`.
- rendezvous: [./rendezvous/EGM.md](./rendezvous/EGM.md) - lease-based rendezvous hashing for work sharding across replicas.
- servicednssigner: [./servicednssigner/EGM.md](./servicednssigner/EGM.md) - signer `servicedns.podcert.ate.dev/identity` for service DNS names.
- podidentitysigner: [./podidentitysigner/EGM.md](./podidentitysigner/EGM.md) - signer `podidentity.podcert.ate.dev/identity` for KSA-equivalent identities.
- postgressigner: [./postgressigner/EGM.md](./postgressigner/EGM.md) - signer `postgres.podcert.ate.dev/identity` for PostgreSQL login certs.
