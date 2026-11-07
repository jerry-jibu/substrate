# boomerutil

## Overview

Shared helpers for the boomer-Go benchmark workers: mutual-TLS dialing of ateapi and the atenet router, lifecycle-failure classification, concurrent-update retries, per-VU goroutine identity, and sampled-span logging with server-side latency extraction.

## Interface

- `client.go:DialControl(endpoint) (*grpc.ClientConn, ateapi.ControlClient, error)` - authenticated gRPC connection to ateapi using the pod certificate.
- `client.go:RouterTLSConfig() (*tls.Config, error)` - mTLS client config for dialing atenet-router; `client.go:DefaultRouterURL` is the default router base URL.
- `lifecycle.go:FailureAction` (`RetryLater`, `ReplaceNow`, `ReplaceIfPersistent`) and `lifecycle.go:ClassifyLifecycleFailure(err)`.
- `lifecycle.go:IsCrashed(err)`, `lifecycle.go:IsConcurrentUpdateConflict(err)`, `lifecycle.go:ConcurrentUpdateMsg`.
- `lifecycle.go:RetryOnConflict(ctx, call)` - up to `ConflictRetryAttempts` (5) attempts.
- `goroutine.go:GoroutineID() int64` - per-VU key via `runtime.Stack`.
- `telemetry.go:MsFloat(d)`, `telemetry.go:LogSampledTrace(span, name, latency, source, err)`, `telemetry.go:ElapsedFromMD(tr, key, fallback)`, `telemetry.go:ElapsedFromHeader(h, key, fallback)`, with `SourceClient`/`SourceServer`.

## Invariants

- Router TLS is TLS 1.3 minimum, pinned to the servicedns CA and the `atenet-router.ate-system.svc` server name, with the pod certificate re-read on every handshake (`client.go:newRouterTLSConfig`).
- `ClassifyLifecycleFailure` maps NotFound, DataLoss, FailedPrecondition, and crashed-Aborted to ReplaceNow; ResourceExhausted, Unavailable, DeadlineExceeded, Canceled, InvalidArgument, PermissionDenied, Unauthenticated, Unimplemented to RetryLater; everything else to ReplaceIfPersistent (`lifecycle.go:ClassifyLifecycleFailure`).
- `IsCrashed` and `IsConcurrentUpdateConflict` are both `codes.Aborted` and distinguished only by message (`lifecycle.go:IsCrashed`, `lifecycle.go:IsConcurrentUpdateConflict`).
- `RetryOnConflict`'s first retry is immediate; later gaps add uniform jitter so race losers do not retry in lockstep (`lifecycle.go:RetryOnConflict`).
- `LogSampledTrace` emits nothing for unsampled spans (`telemetry.go:LogSampledTrace`).

## Side Effects

- Dials ateapi with in-cluster or kubeconfig credentials, an ateapiauth dial option set, a k8sresolver resolver, and an otelgrpc stats handler (`client.go:DialControl`).
- Reads projected files `/run/servicedns-ca/ca.crt` and `/run/podidentity.podcert.ate.dev/credential-bundle.pem` (`client.go` constants).
- Emits structured log lines for sampled spans (`telemetry.go:LogSampledTrace`).

## Verification

- Run: `go test ./internal/benchmarking/boomer/boomerutil/...`
