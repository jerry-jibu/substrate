# testtypes

One module per `tests.yaml` `type`; `orchestrator.py` dispatches via the
package's `TYPES` registry. Each module implements the TestType hooks called in
lifecycle order: `validate`, `build_image`, `pre_test`, `job_tmpl`, `job_subs`.

## Interface

- `__init__.py:TestType` - the Protocol every test type implements: `TEST_TYPE` plus the five hooks.
- `__init__.py:TYPES` - registry mapping `"locust"` and `"nighthawk-ingress"` to their modules; adding a test type is one new module plus one entry here.
- `locust.py:validate` - requires `file`, `duration`, `users`; validates a `controlPlane:` block through `controlplane.validate`.
- `locust.py:build_image` - builds and pushes the locust runner image from `benchmarking/locust/Dockerfile` to `${KO_DOCKER_REPO}/locust-test:<commit>`.
- `locust.py:pre_test` - with a `controlPlane:` block, shapes the control plane via `controlplane.shape`; without it, leaves the cluster as installed.
- `locust.py:job_tmpl` / `locust.py:job_subs` - the shared runner Job template and its `${...}` substitutions (`TEST_FILE`, `DURATION`, `USERS`, `RUNNER_CPU` default `500m`, `RUNNER_MEMORY` default `512Mi`).
- `nighthawk_ingress.py:DEFAULTS` / `nighthawk_ingress.py:config` - the `nighthawk-ingress:` block defaults (client sizing, search parameters, SLO thresholds) merged with the test's block; `envoyCpu` has no default.
- `nighthawk_ingress.py:validate` - requires `duration`, `workerCount` (int >= 1, also the actor fleet size), `envoyCpu` (int >= 1), a valid `clientConcurrency`, parseable durations, and a DNS-label `atespace`; unknown knobs are rejected, not merged.
- `nighthawk_ingress.py:build_image` - builds and pushes the nighthawk runner image from `benchmarking/nighthawk-ingress/Dockerfile` to `${KO_DOCKER_REPO}/nighthawk-ingress-test:<commit>`.
- `nighthawk_ingress.py:pre_test` - pins `atenet-router` to the test's `envoyCpu`: both containers get requests=limits (Guaranteed QoS) and envoy's command gains `--concurrency <cpu>`, dropping the base manifest's debug flags; blocks on rollout.
- `nighthawk_ingress.py:job_subs` - the nighthawk Job substitutions, including one actor per worker (`ACTORS` = `workerCount`) for a warm-path-only measurement.

## Invariants

- `__init__.py` asserts at import that every registered type has all five hooks, so a broken type fails at every entry point instead of mid-benchmark (`__init__.py` module tail).
- `locust.py:build_image` and `nighthawk_ingress.py:build_image` require `KO_DOCKER_REPO` in the environment and push to the same project as the test cluster so the runner Job can pull (`os.environ['KO_DOCKER_REPO']`).
- Unknown nighthawk knobs fail validation rather than silently running with defaults (`nighthawk_ingress.py:validate`).
- The nighthawk atespace defaults to one fixed `ingress-benchmark`, never per run, because nothing deletes atespaces automatically (`nighthawk_ingress.py:DEFAULTS`).
- Nighthawk client sizing is decoupled from `envoyCpu` so the harness never binds (`nighthawk_ingress.py:DEFAULTS`).

## Side Effects

- `build_image` runs `docker build --platform linux/amd64` and `docker push` (`util.py:build_and_push`).
- `nighthawk_ingress.py:pre_test` patches the `atenet-router` Deployment in `ate-system` and waits for its rollout; there is no unpatch, since every test redeploys substrate (`nighthawk_ingress.py:pre_test`).
- `locust.py:pre_test` mutates the cluster only through `controlplane.shape` (`locust.py:pre_test`).

## Verification

- Covered by `../test_locust.py` and the other unit tests next to the orchestrator; no test runner is pinned in this directory.
