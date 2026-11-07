# nighthawk-egress

Kubernetes manifests and deploy scripts for the egress benchmark: an
in-cluster `nighthawk_test_server` origin, a `traffic-driver` actor (Envoy
`nighthawk_service` inside a gVisor sandbox) driven through `atenet-router`,
and an Envoy invoker proxy that injects the target-actor header on the
driver's requests.

## Interface

- `deploy.sh` - applies `namespace.yaml`, `nighthawk-server.yaml`, and `traffic-driver-invoker.yaml`, then waits for both Deployments; `--delete` removes them. `--context` and `--wait-timeout` (default 120 s).
- `deploy-traffic-driver.sh` - deploys the actor side: resolves `traffic-driver.yaml.tmpl` with ko, creates the `nighthawk` atespace, the `traffic-driver` ActorTemplate (waiting for its golden snapshot), the `tester` actor, its EgressPolicy, and resumes it. `--delete` tears it down. `--ko-docker-repo` (default `KO_DOCKER_REPO` or `localhost:5001`), `--wait-timeout` (default 300 s).
- `namespace.yaml` - the `nighthawk` namespace both sides deploy into.
- `nighthawk-server.yaml` - ConfigMap + Deployment + Service: `nighthawk_test_server` on ports 80/8080 (plaintext) and 443/8443 (TLS via a `servicedns.podcert.ate.dev/identity` serving cert with SDS watched-directory reload), pinned by digest.
- `traffic-driver.yaml.tmpl` - `ate.dev/v1` WorkerPool `traffic-driver`, one replica, gVisor worker image.
- `traffic-driver-template.yaml.tmpl` - protojson-shaped `ateapi.ActorTemplate` (not a CRD): runs `nighthawk_service --listen 0.0.0.0:80 --service traffic-generator-service` with the egress trust bundle mounted, 2000 millicores / 1 GiB, `SNAPSHOT_FIDELITY_MEMORY`.
- `traffic-driver-invoker.yaml` - Envoy proxy on :19980 that injects `ate-target-actor: nighthawk/tester` and forwards upstream over mTLS to `atenet-router.ate-system.svc.cluster.local:443`.
- `egress-policy.yaml` - protojson-shaped `ateapi.EgressPolicy` allowing http on 80, https on 443, and TLS passthrough on 8443 to any hostname.

## Invariants

- The ActorTemplate and EgressPolicy are substrate resources applied through `kubectl-ate`, not Kubernetes CRDs (`traffic-driver-template.yaml.tmpl` and `egress-policy.yaml` header comments).
- The template's golden snapshot must be ready before the actor is created from it (`deploy-traffic-driver.sh:wait_template_ready`).
- `deploy-traffic-driver.sh:deploy` removes any existing actor and template before redeploying, so a rerun starts from a clean state.
- `nighthawk-server.yaml`, `traffic-driver-template.yaml.tmpl`, and `benchmarking/nighthawk-ingress/Dockerfile` pin the same `envoyproxy/nighthawk-dev` digest (`nighthawk-server.yaml` image, `traffic-driver-template.yaml.tmpl` image).
- The server's TLS serving cert SAN is `nighthawk-server.nighthawk.svc`, minted only because a selecting Service exists (`nighthawk-server.yaml` Service comment).

## Side Effects

- Both scripts run `kubectl apply`/`delete` against a cluster; `deploy-traffic-driver.sh` also runs `ko resolve` and `kubectl-ate` create/suspend/delete/resume on the `nighthawk` atespace (`deploy.sh:deploy`, `deploy-traffic-driver.sh:deploy`).

## Verification

- Run: `hack/verify/shellcheck.sh` (shellchecks every repo `.sh` script).
