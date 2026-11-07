# ate-setup/internal/demos

## Overview

Holds the contract every bundled demo implements, the registry the command tree
is built from, and the shared behavior of a demo that is one manifest template.
Each demo lives in its own package under this one and registers itself from
`init`, so adding a demo means adding a directory and one import rather than
editing a shared switch, and no demo can reach into another's placeholders,
namespaces, or template (`demos.go` package doc).

## Interface

- `Demo` (`demos.go`) - the demo contract: `Name`, `Description`, `Settings`, `Deploy`, `Delete`.
- `Register` (`demos.go`) - adds a demo to the registry and registers its settings against its deploy command path; panics on a duplicate name.
- `All`, `Deleters`, `DeployCommandPath` (`demos.go`) - registry access for the command tree and `steps.Env.DeleteAll`.
- `Substrate`, `SubstrateTemplate` (`substrate.go`) - the shared substrate-resource demo shape: one WorkerPool manifest plus ActorTemplates created through the ate API, with deploy-time rollouts and golden-snapshot waits.
- `MicroVMGoldenTimeout` (`substrate.go`) - the 600s golden budget micro-VM demos set.
- `Render` (`render.go`) - expands a demo template with values through `render.Template`.
- `ExternalVolumePlaceholders` (`render.go`) - the optional external-volume hooks in the counter templates, dropped when no value supplies them.
- Subdirectory contracts:
  - [all](./all/EGM.md) - links every bundled demo into the binary.
  - [claudemultiplex](./claudemultiplex/EGM.md) - several Claude Code agents on one WorkerPool.
  - [counter](./counter/EGM.md) - counter actor with snapshot, resume, ingress, and optional external volume.
  - [countermicrovm](./countermicrovm/EGM.md) - counter demo on microVM workers.
  - [demotest](./demotest/EGM.md) - shared render-test fixtures.
  - [egress](./egress/EGM.md) - egress policy enforcement through atenet.
  - [egressmicrovm](./egressmicrovm/EGM.md) - egress demo on microVM workers.
  - [jupyter](./jupyter/EGM.md) - unmodified Jupyter notebook image as a suspending actor.
  - [multitemplate](./multitemplate/EGM.md) - two ActorTemplates sharing one WorkerPool.
  - [parking](./parking/EGM.md) - actor parking and unparking on a small WorkerPool.
  - [sandbox](./sandbox/EGM.md) - on-demand sandbox actor driven by the sandbox client.

## Invariants

- `Delete` must succeed on a cluster where the demo was never installed, because DeleteAll runs every demo's Delete (`demos.go` `Demo.Delete`).
- No demo can reach into another's placeholders, namespaces, or template; each owns its own package (`demos.go` package doc).
- A demo's settings are registered where the demo package is linked, bound as flags only on that demo's deploy command (`demos.go` `Register`).
- `Substrate.Deploy` blocks until each template's golden snapshot exists, so callers run against an already-warm node instead of racing cold-start work (`substrate.go` `Deploy`).

## Side Effects

- `Substrate.Deploy` applies manifests to the cluster (`e.ResolveAndApplyBytes`), waits for pool Deployments (`kube.RolloutStatus`), creates atespaces and ActorTemplates through the ate API client, and waits for golden snapshots (`steps.WaitActorTemplateGolden`) (`substrate.go`).
- `Substrate.Delete` deletes template refs and atespaces, deletes the rendered pool objects, and returns left-over namespace names for the caller to wait on (`substrate.go` `Delete`).
- `Register` mutates the package-level registry map at init time (`demos.go`).

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/...`
