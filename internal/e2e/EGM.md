# e2e

## Overview

Shared harness for Agent Substrate's end-to-end tests (`hack/run-e2e.sh`): client setup, namespace lifecycle, fixture/template deployment, router and statusz clients, sandbox and dataplane selection, manifest rendering, and small retry/wait helpers. Per-suite tests live under `suites/` (each with its own `testmain_test.go` calling `RunTestMain`) and `upgrade/`; they are not documented here.

## Interface

- Entry: `testmain.go:RunTestMain(m *testing.M) int` and `testmain.go:RegisterSuiteCleanup(fn)`; `preflight.go:PreflightChecks() error`.
- Clients: `clients.go:Clients` and `clients.go:GetClients()` / `clients.go:NewClients(ctx)` / `Clients.Close()`.
- Environment: `env.go:CheckEnv(keys...)`, `env.go:SystemNamespace()`, `env.go:ResourceName(name)`; `sandbox.go:SandboxClass()`/`IsMicroVM()`/`IsAgentGateway()`, `sandbox.go:SubstrateCounterFixture()`, `sandbox.go:EgressFixture()`, `sandbox.go:RenderFixtureManifest(t, relPath, name)`; `atenet_dataplane.go:CurrentAtenetDataplane() AtenetDataplane`; `backend.go:RequireGCSBackend(t, backend)`; `snapshotdelays.go:RequireShortSnapshotDelays(t, ctx, maxDelay)`.
- Namespaces: `namespace.go:CreateNamespace(t)` (labeled `namespace.go:NamespaceLabel` "ate.dev/e2e"), `namespace.go:RetainNamespaces()`, `namespace.go:CleanupNamespaces()`.
- Fixtures and templates: `fixture.go:DeploySubstrateFixture(t, ctx, clients, manifests, name, trustBundles, opts...)` with `fixture.go:WithTemplateValue`/`WithImageUser`; `probe.go:DeployProbe(t, name, opts...)` with `WithTrustBundle`/`WithSystemRoots` (`probe.go:ProbeName` = "probe"); `template.go:CreateSubstrateTemplateFrom` / `CreateSubstrateCounterTemplate` / `WaitForSubstrateTemplateReady` / `TemplateRef`; `manifest.go:ResolveFixtureImage(t, importPath)`.
- Actor helpers: `resume.go:ResumeActorAwaitCapacity(t, ctx, clients, req)`; `retry.go:RetryAborted(t, ctx, rpc, req)`; `egresspolicy.go:EnsureEgressPolicy(t, ctx, clients, actor, rules...)` plus the `EgressAllow*`/`EgressInjectHeader*` rule builders; `sandboxkill.go:KillActorSandbox(...)`, `sandboxkill.go:ReadWorkerProcessStatus(...)`; `serverpod.go:DeployServerPod`/`CreateServerService`/`WaitForServerEndpoint` and `serverpod.go:ServiceDNSTLS(spec)`; `pod.go:WaitForPodReady`, `pod.go:DescribePodState`.
- Clients for the data plane: `router_client.go:NewRouterClient(t, ctx)` with `Get`/`PostJSON`/`Do`/`Connect`/`ConnectWithHeaders`/`ConnectPortForward`; `statusz.go:NewStatuszClient(ctx)` with `Parking`, `WaitForCount`, `Close`; `ingressauth.go:RouterClientCertificate(t, ctx, serviceAccount)`, `ingressauth.go:RouterTLSConfig`, `ingressauth.go:RouterClientSPIFFEID(sa)`.
- Trust bundles: `trustbundle.go:EnsureEgressTrustBundle`, `trustbundle.go:AddEgressCA`, `trustbundle.go:SameCertificates`, `trustbundle.go:ServiceDNSTrustBundle`.
- Metrics: `collector_metrics.go:ScrapeCollectorMetrics` / `ScrapeAgentGatewayRouterMetrics` / `ScrapeEgressEnvoyMetrics` / `MissingPlatformMetrics` / `LifecycleEventSeries` / `StatesNotAdvanced` / `CollectorHasService` / `TargetInfoLabel`; `credprovider.go:ConfigureCredentialProvider(t)`.
- Output helpers: `pretty.go:Colorf`, `pretty.go:ColorWriter`, `pretty.go:NewIndentWriter`; process helpers `run.go:FindRepoRoot`, `run.go:RunCmd`/`RunCmdOutput`/`RunCmdWithEnv`.
- Subdirectory contracts:
  - [fixtures](./fixtures/EGM.md) - probe and testserver workload fixtures and their YAML templates.
  - [procstatus](./procstatus/EGM.md) - the /proc-derived JSON types worker security tests compare.

## Invariants

- Every suite runs through `RunTestMain`, which performs `PreflightChecks`, namespace setup/teardown, and registered cleanups (`testmain.go:RunTestMain`).
- E2E namespaces carry the label `ate.dev/e2e` so cleanup can find them (`namespace.go:NamespaceLabel`).
- `RetryAborted` retries `codes.Aborted` up to a 30s bound between attempts (`retry.go:abortedRetryWait`); `ResumeActorAwaitCapacity` waits up to 90s for capacity (`resume.go:resumeCapacityWait`).
- The egress CA pool is only ever written through atecontroller's `EgressMITMTrustReconciler` Secret; suites provision the pool and let the reconciler publish the bundle, and `AddEgressCA` waits until the reconciler's bundle actually contains the new CA (`trustbundle.go` package doc, `trustbundle.go:AddEgressCA`).
- Sandboxed fixture selection branches on `E2E_SANDBOX_CLASS` (gVisor default vs `microvm`) and the dataplane on `E2E_ATENET_DATAPLANE` (envoy vs agentgateway), so one suite covers both runtimes (`sandbox.go`, `atenet_dataplane.go:AtenetDataplaneEnv`).
- Manifest templates carry `${...}` placeholders rendered by `RenderFixtureManifest`; fixture images resolve through the repo's pinned ko so CI need not install it (`manifest.go` doc, `manifest.go:ResolveFixtureImage`).

## Side Effects

- Creates and deletes Kubernetes namespaces, WorkerPool CRDs, ActorTemplates, Actors, EgressPolicies, Services, and pods through the ateapi and Kubernetes clients (`namespace.go`, `template.go`, `fixture.go`, `serverpod.go`).
- Writes egress CA pool Secrets (consumed by atecontroller's reconciler) and reads published bundles (`trustbundle.go`).
- Port-forwards to the atenet router's status port and the router service (`statusz.go:NewStatuszClient`, `router_client.go:ConnectPortForward`).
- Runs subprocesses (`run.go:RunCmd*`) and scrapes metrics endpoints over HTTP (`collector_metrics.go`).

## Verification

- Run: `go test ./internal/e2e/...` (full suites require a cluster with Agent Substrate installed; `hack/run-e2e.sh` is the entry point)
