# contextlogging

## Overview
A `slog.Handler` wrapper installed as the default logger in every server (via serverboot). It stamps trace-context fields onto records and redacts `debug_redact` protobuf fields wherever they are logged in the process.

## Interface
- `contextlogging.go:NewHandler` - wraps an inner `slog.Handler`.
- `contextlogging.go:ContextHandler` - the handler; `Handle`, `Enabled`, `WithAttrs`, `WithGroup` implement `slog.Handler`.

## Invariants
- Trace fields use the OTel-spec spellings for non-OTLP log formats, gated on the whole span context being valid: a trace ID without a span ID names a request but not the operation within it (`contextlogging.go:Handle`).
- Redaction of `debug_redact` fields happens here, in the shared handler, so it also covers protos logged anywhere else in the process (`contextlogging.go:redactRecord`).
- Pre-bound attributes are redacted once at bind time in `WithAttrs`, since `Handle` never sees them again (`contextlogging.go:WithAttrs`).
- Records are never edited in place: a copy is started at the first changed attribute (`contextlogging.go:redactRecord`).

## Side Effects
- Delegates writing to the inner handler; no direct I/O of its own.

## Verification
- Run: `go test ./internal/contextlogging/...`
