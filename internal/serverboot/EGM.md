# serverboot

## Overview

Startup boilerplate shared by the long-running substrate server binaries
(ateapi, atelet, the workers): slog wiring, OTel tracer/meter/logger
providers with relay or direct export, a Prometheus `/metrics` server, a
`/readyz` and `/healthz` server for the kubelet probes, and fail-fast
helpers.

## Interface

- `serverboot.go:InitLogger`/`InitLoggerWithWriter` - global JSON slog handler wrapped in `contextlogging.NewHandler`; the writer variant shares one synchronized writer with actor-log forwarding.
- `serverboot.go:SetLogLevel` - dynamic level from `debug|info|warn|error`; empty leaves it unchanged.
- `serverboot.go:InitTracing` - global TracerProvider; `serverboot.go:TracingOptions` carries `ServiceName`, `Sampling`, `ExporterConn` (relay connection), `RelayCapable`.
- `sampling.go:TraceSampling`, `ParentRatioSampling`, `ParentNeverSampling`, `ResolveTraceSampling` - sampling policy with `RootSamplingPercent`; `sampling.go:ControlPlaneTraceRatio` (0.1); `OTEL_TRACES_SAMPLER`/`_ARG` override the default and invalid values keep it with a warning.
- `serverboot.go:InitMetrics` - MeterProvider with a Prometheus reader plus OTLP push while selected; `serverboot.go:InitMetricsBridged` - for a binary whose `/metrics` serves a registry it does not own (`skipFamilies` avoids double push); `serverboot.go:InitMetricsPushOnlyVia` - push-only via a relay conn, implies relay-capable.
- `exporters.go:Exporters`/`Has`/`resolveExportersEnv` - `OTEL_*_EXPORTER` parsing (`exporters.go:ExporterOTLP`, `exporters.go:ExporterConsole`, `exporters.go:exporterPrometheus`, `exporters.go:exporterNone`); unknown items are skipped, `none` with an exporter is rejected, invalid values fall back to the default with a warning.
- `logging.go:ResolveLogsExporter`/`LoggingOptions`/`InitLogging` - LoggerProvider for actor-event records; default exporter set is none; returns `(nil, nil)` when nothing exports.
- `serverboot.go:Readiness`/`MarkNotReady`/`AddCheck` - readiness predicate; `/readyz` 503 while draining or a check fails within `serverboot.go:checkTimeout` (500ms); `/healthz` always 200.
- `serverboot.go:StartMetricsServer`/`StartHealthServer` - HTTP servers; a serve failure exits the process.
- `serverboot.go:Fatal` - log error and exit 1; `serverboot.go:ShutdownProvider` - deferred provider shutdown with logged errors.
- `serverboot.go:otlpExportBlocked`/`hasOTLPDestination` - decide and explain when a signal has nowhere to export.

## Invariants

- OTLP export is insecure on the direct path (GKE managed traces/metrics cannot validate collector TLS); `WithGRPCConn` takes precedence over endpoint options (`serverboot.go:InitTracing`, `newMeterProvider` comments).
- The exporter connection is owned by the caller; provider Shutdown does not close it (`serverboot.go:TracingOptions.ExporterConn`, `InitMetricsPushOnlyVia` docs).
- A relay-capable component without a relay conn needs an explicit OTLP endpoint (the SDK localhost default reaches nothing on a worker pod), else the signal is dropped with a Warn-level reason (`serverboot.go:hasOTLPDestination`, `otlpExportBlocked`).
- Sampling env is resolved by serverboot, not the SDK, because an explicit WithSampler silences SDK env handling and the SDK falls open to 100% on invalid values; here invalid values keep the component default and log (`sampling.go:ResolveTraceSampling`).
- Set-but-empty env vars read as unset, because templated manifests render empty vars (`sampling.go:resolveTraceSampling`, `exporters.go:resolveExporters`).
- `MarkNotReady` is permanent; readiness checks affect `/readyz` only, never liveness (`serverboot.go:Readiness` doc).
- `dropRPCServerAddress` removes `server.address` from otelgrpc metrics because the socket path or pod IP makes it one value per pod (`serverboot.go:dropRPCServerAddress`).
- Telemetry resource: `WithFromEnv` runs last so `OTEL_*` env vars override defaults (`serverboot.go:newResource`).

## Side Effects

- Serves `/metrics`, `/readyz`, `/healthz` HTTP endpoints (`serverboot.go:StartMetricsServer`, `serverboot.go:StartHealthServer`).
- Registers global OTel providers and propagator, and dials the collector over gRPC when exporting (`serverboot.go:InitTracing`, `newMeterProvider`, `logging.go:InitLogging`).
- Reads `OTEL_*` environment variables (samplers, exporters, endpoints).
- Exits the process on startup failures (`serverboot.go:Fatal`, serve failures).

## Verification

- Run: `go test ./internal/serverboot/...`
