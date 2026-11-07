# internal

## Overview

Shared packages internal to this module: node and sandbox plumbing shared
by the binaries in `cmd/`, the atelet/worker protocol code, and the
control-plane helpers. Only `internal/proto` and the paths in
`internal/nodepath` are shared with the worker runtimes; everything else
stays module-private. Every child below has a contract file.

## Subdirectories & Boundaries

- actoridjwt: [./actoridjwt/EGM.md](./actoridjwt/EGM.md) - actor-id JWT claim types (`actoridjwt.WireClaims`, `Kind`) decoded by `oidcjwt`.
- actorlock: [./actorlock/EGM.md](./actorlock/EGM.md) - per-actor lifecycle locks and in-flight RPC tracking (`actorlock.Locks`, `InFlight`).
- actorlog: [./actorlog/EGM.md](./actorlog/EGM.md) - forwards actor stdout/stderr to pod logs as `ate.dev/*`-labeled JSON (`actorlog.ActorLogger`, `NewSyncedWriter`).
- actorevent: [./actorevent/EGM.md](./actorevent/EGM.md) - actor lifecycle event records; `actorevent.SetConsole` selects the stdout form.
- ateattr: [./ateattr/EGM.md](./ateattr/EGM.md) - shared log/metric attribute keys (`ateattr.ActorUIDKey`, `StatsMemoryUsageKey`).
- ateclient: [./ateclient/EGM.md](./ateclient/EGM.md) - shared ate-api client plumbing.
- ateinterceptors: [./ateinterceptors/EGM.md](./ateinterceptors/EGM.md) - gRPC server interceptors (`InternalServerUnaryInterceptor`).
- atelet: [./atelet/EGM.md](./atelet/EGM.md) - atelet-side library code (`atelet.DefaultPort`).
- ateletdial: [./ateletdial/EGM.md](./ateletdial/EGM.md) - dialing atelet from the control plane.
- atenet: [./atenet/EGM.md](./atenet/EGM.md) - atenet daemon library packages.
- ateapiauth: [./ateapiauth/EGM.md](./ateapiauth/EGM.md) - mTLS dial options for ate-api-server (`ateapiauth.DialOptions`, `ClientConfig`).
- apiconfig: [./apiconfig/EGM.md](./apiconfig/EGM.md) - API server configuration.
- apierror: [./apierror/EGM.md](./apierror/EGM.md) - canonical gRPC error constructors (`apierror.Unavailable`, `NotFound`, `ResourceExhausted`, `InvalidArgument`, `FailedPrecondition`, `Internal`).
- atunnel: [./atunnel/EGM.md](./atunnel/EGM.md) - actor egress tunnel and DNS policy (`atunnel.Start`, `Tunnel.ActivateEgress`/`ActivateIngress`, `RegisterFlags`).
- ateworker: [./ateworker/EGM.md](./ateworker/EGM.md) - worker-side libraries: `actornet` (sandbox networks, `ActorVethIP`), `cgroup`, `memborrow`, `pid1`, `phaselog`, `register` (capacity reporting, epochs), checkpoint helpers.
- ateworkervalidation: [./ateworkervalidation/EGM.md](./ateworkervalidation/EGM.md) - worker RPC request validation (`ValidateRunWorkloadRequest` and peers).
- benchmarking: [./benchmarking/EGM.md](./benchmarking/EGM.md) - benchmark libraries (`glutton`, `boomer` incl. `boomerutil.DialControl`, `dynconfig`).
- bootid: [./bootid/EGM.md](./bootid/EGM.md) - node boot id (`bootid.Read`, `bootid.Path`).
- cdi: [./cdi/EGM.md](./cdi/EGM.md) - Container Device Interface support.
- clustertrustbundle: [./clustertrustbundle/EGM.md](./clustertrustbundle/EGM.md) - Kubernetes ClusterTrustBundle projection.
- contextlogging: [./contextlogging/EGM.md](./contextlogging/EGM.md) - slog handler carrying context values (`contextlogging.NewHandler`).
- credbundle: [./credbundle/EGM.md](./credbundle/EGM.md) - credential/trust bundle loaders (`credbundle.Loader`, `ClientLoader`, `PoolLoader`).
- deviceplugin: [./deviceplugin/EGM.md](./deviceplugin/EGM.md) - Kubernetes device plugin integration.
- egresspolicy: [./egresspolicy/EGM.md](./egresspolicy/EGM.md) - egress policy model (`HostnamePattern.Matches`, mirrored by the Envoy module's `pattern_matches`).
- e2e: [./e2e/EGM.md](./e2e/EGM.md) - end-to-end test harness packages.
- hardware: [./hardware/EGM.md](./hardware/EGM.md) - host compatibility probing (`hardware.ProbeHost`).
- imagecache: [./imagecache/EGM.md](./imagecache/EGM.md) - node image cache (`imagecache.SetupBundleRootfs`, `ImageVolumeMountPath`, `UnmountAllUnder`).
- installdefaults: [./installdefaults/EGM.md](./installdefaults/EGM.md) - install-time constants (`installdefaults.SPIFFEID`).
- k8sresolver: [./k8sresolver/EGM.md](./k8sresolver/EGM.md) - gRPC name resolution against Kubernetes EndpointSlices.
- localca: [./localca/EGM.md](./localca/EGM.md) - throwaway CA issuing (`localca.GenerateCA`, `localca.ConcretePool`).
- localjwtauthority: [./localjwtauthority/EGM.md](./localjwtauthority/EGM.md) - actor JWT signing authority.
- linuxsys: [./linuxsys/EGM.md](./linuxsys/EGM.md) - Linux system call helpers.
- netns: [./netns/EGM.md](./netns/EGM.md) - network namespace creation, execution, dialing, sysctls.
- nodepath: [./nodepath/EGM.md](./nodepath/EGM.md) - node-level shared paths and socket names.
- objectstore: [./objectstore/EGM.md](./objectstore/EGM.md) - object storage helpers (`objectstore.BucketPrefix`).
- objectstoreplugin: [./objectstoreplugin/EGM.md](./objectstoreplugin/EGM.md) - object-store plugin client plumbing.
- ocispec: [./ocispec/EGM.md](./ocispec/EGM.md) - OCI spec build and per-runtime shaping.
- oidcjwt: [./oidcjwt/EGM.md](./oidcjwt/EGM.md) - OIDC JWT verification.
- otlprelay: [./otlprelay/EGM.md](./otlprelay/EGM.md) - node-local OTLP relay (atelet server, worker dialer).
- pemutil: [./pemutil/EGM.md](./pemutil/EGM.md) - PEM bundle sanitization for projection.
- pidfd: [./pidfd/EGM.md](./pidfd/EGM.md) - process-exit watching via pidfd.
- pluginsocket: [./pluginsocket/EGM.md](./pluginsocket/EGM.md) - server-to-plugin unix socket transport.
- podidentity: [./podidentity/EGM.md](./podidentity/EGM.md) - PodCertificateRequest client for out-of-pod identities.
- portforward: [./portforward/EGM.md](./portforward/EGM.md) - Service-resolved Kubernetes port forwarding.
- preview: [./preview/EGM.md](./preview/EGM.md) - `--preview` feature gates.
- principal: [./principal/EGM.md](./principal/EGM.md) - authenticated principal context type.
- proto: [./proto/EGM.md](./proto/EGM.md) - the atelet/ateworker wire protos (`ateletpb`, `ateworkerpb`).
- protoredact: [./protoredact/EGM.md](./protoredact/EGM.md) - masks `[debug_redact]` proto fields for logging.
- resources: [./resources/EGM.md](./resources/EGM.md) - refs, snapshot URIs, SPIFFE IDs, validation, capacity math.
- roottest: [./roottest/EGM.md](./roottest/EGM.md) - root-privilege test gate.
- sandboxruntimes: [./sandboxruntimes/EGM.md](./sandboxruntimes/EGM.md) - worker sandbox runtimes file.
- serverboot: [./serverboot/EGM.md](./serverboot/EGM.md) - server startup boilerplate (logging, OTel, probes).
- sizing: [./sizing/EGM.md](./sizing/EGM.md) - sandbox sizing from declared limits.
- snapshotcache: [./snapshotcache/EGM.md](./snapshotcache/EGM.md) - snapshot file drop and page-cache eviction.
- substratex509: [./substratex509/EGM.md](./substratex509/EGM.md) - identity X.509 extensions.
- tarutil: [./tarutil/EGM.md](./tarutil/EGM.md) - metadata-preserving tar archive/restore.
- testcert: [./testcert/EGM.md](./testcert/EGM.md) - test TLS certificate issuance.
- testenv: [./testenv/EGM.md](./testenv/EGM.md) - envtest control plane for apiserver-backed tests.
- version: [./version/EGM.md](./version/EGM.md) - build identity.
- volume: [./volume/EGM.md](./volume/EGM.md) - volume types and staging helpers.
- volumepath: [./volumepath/EGM.md](./volumepath/EGM.md) - projected volume path validation.
- wakeupprobe: [./wakeupprobe/EGM.md](./wakeupprobe/EGM.md) - container HTTP readiness polling.
- workercapacity: [./workercapacity/EGM.md](./workercapacity/EGM.md) - downward-API capacity file names.
- workerstats: [./workerstats/EGM.md](./workerstats/EGM.md) - worker usage sampling and records (`workerstats.Activation`, `StartSampler`, `UsageEmitter`).

## Global Invariants

- atelet and the worker runtimes share only `internal/proto` and `internal/nodepath` (AGENTS.md); the rest of this tree is free to change with either side.
- `internal/proto/ateworkerpb` and `internal/proto/ateletpb` are generated; regenerate rather than hand-edit.

## Global Side Effects

- ["None"] (the individual packages' effects are in their contracts)

## System Verification

- Run: `go test ./internal/...`
