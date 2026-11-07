# ateinterceptors

## Overview
gRPC server interceptors shared by the ate services: request/response logging with a server-elapsed trailer, deadline capping, internal-service error normalization, and rejection of protobuf requests carrying fields this binary does not know.

## Interface
- `ateinterceptors.go:ServerUnaryInterceptor` - logs request and response, sets the `x-server-elapsed-us` trailer.
- `ateinterceptors.go:MaxDeadlineUnaryInterceptor` - returns an interceptor capping the request context at `maxDeadline`.
- `ateinterceptors.go:InternalServerUnaryInterceptor` - for internal services; handler errors keep their apierror code, all others become Internal.
- `ateinterceptors.go:ServerElapsedTrailer` - trailer constant carrying handler duration in microseconds.
- `rejectunknown.go:RejectUnknownFieldsUnaryInterceptor` - refuses any request carrying an unknown protobuf field at any nesting level.

## Invariants
- Redaction of `debug_redact` fields happens in the shared slog handler installed by serverboot, not in the interceptor; the interceptor must not mutate the logged or returned messages (`ateinterceptors.go:ServerUnaryInterceptor`).
- `MaxDeadlineUnaryInterceptor` caps the context but preserves a caller's shorter deadline (`ateinterceptors.go:MaxDeadlineUnaryInterceptor`).
- `InternalServerUnaryInterceptor` never forwards an upstream service's status code; anything but an apierror from the handler reaches the caller as Internal with its full text (`ateinterceptors.go:InternalServerUnaryInterceptor`).
- List responses are elided from the log; a failed call's response logs as the handler returned it (`ateinterceptors.go:respForLog`).
- Unknown-field rejection reports every unknown field at every nesting level, path-qualified (`rejectunknown.go:findUnknownFields`).

## Side Effects
- Writes request/response log lines through the default logger (`ateinterceptors.go:ServerUnaryInterceptor`).
- Sets response trailers (`ateinterceptors.go:ServerElapsedTrailer`).

## Verification
- Run: `go test ./internal/ateinterceptors/...`
