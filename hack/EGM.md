# hack

Dev and CI scripts for Agent Substrate: every verification, code
generation, cluster bootstrap, and e2e entry point lives here as a
standalone shell script so `shellcheck` can lint it. Top-level scripts
aggregate the per-directory scripts under the subdirectories below.

## Interface

- `verify-all.sh` - runs every verifier even if one fails, so a single run reports all failures.
- `run-e2e.sh` - runs the e2e tests against a cluster with Agent Substrate installed.
- `run-e2e-kind.sh` - wraps `run-e2e.sh` with the Kind-specific environment for a local cluster.
- `run-root-tests.sh` - runs every root-gated test package under sudo; packages importing `internal/roottest` self-skip unless run as root.
- `update-all.sh` / `update-tool.sh` / `run-tool.sh` - run all code generators, or update/run one tool from `hack/tools/`.
- `create-kind-cluster.sh` / `delete-kind-cluster.sh` / `kind.sh` / `setup-csi-nfs-kind.sh` / `setup-e2e-egress-tls-kind.sh` - local cluster lifecycle for dev and e2e.
- `protoc.sh`, `site.sh`, `test-site-templates.sh`, `test-dynamic-modules.sh`, `cleanup-e2e.sh`, `e2e-focus-sandbox-class.sh`, `verify-atenet-drain.sh`, `verify-egress-demo.sh` - focused dev/CI helpers.
- Subdirectories (each has a contract): [./metrics/EGM.md](./metrics/EGM.md), [./release/EGM.md](./release/EGM.md), [./update/EGM.md](./update/EGM.md), [./util/EGM.md](./util/EGM.md), [./verify/EGM.md](./verify/EGM.md).

## Invariants

- Every script is a standalone `.sh` file so `shellcheck` can lint it; workflow YAML and other languages must call these scripts rather than inlining shell (AGENTS.md "Code" section).
- Each script under `./verify/` is one check, and `verify-all.sh` must keep an entry per script (`./verify/EGM.md`).

## Side Effects

- The cluster scripts create and delete Kind clusters and apply Kubernetes resources on the local machine (`create-kind-cluster.sh`, `delete-kind-cluster.sh`, `run-e2e-kind.sh`).
- `run-root-tests.sh` executes test binaries under `sudo` (`run-root-tests.sh` header).
- CI workflows write test artifacts into `_artifacts/` (gitignored); `verify-all.sh` itself only runs the verifiers and reports failures (`verify-all.sh:17-39`).

## Verification

- Run: `./hack/verify-all.sh` (full local verification); `./hack/run-root-tests.sh` for root-gated packages. The scripts themselves are linted by `verify-all.sh`'s shellcheck step.
