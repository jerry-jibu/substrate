# apierror

## Overview
Constructor set for gRPC-scoped errors. Each constructor fixes the `codes.Code` a failure reaches the caller with and documents when to use it; `FromError` and `Code` convert any error into the status a server returns.

## Interface
- `apierror.go:InvalidArgument`, `NotFound`, `AlreadyExists`, `FailedPrecondition`, `Aborted`, `PermissionDenied`, `Unauthenticated`, `ResourceExhausted`, `Unimplemented`, `Unavailable`, `DataLoss`, `Internal` - error constructors, printf-style.
- `apierror.go:Code` - the code `err` reaches the caller with.
- `apierror.go:FromError` - converts `err` to the `*status.Status` a server returns, with `ok` reporting whether `err` was an apierror or context error.

## Invariants
- `Code` returns `codes.OK` for nil and `codes.Internal` for an error that is neither an apierror nor a context error; it never reads a gRPC status (`apierror.go:Code`).
- `Unavailable` must never be used for a failure that leaves partial state behind (`apierror.go:Unavailable`).
- Wrapping preserves `errors.Is`/`Unwrap` reachability of the cause (`apierror.go:Unwrap`).
- `FromError` uses the apierror's own message, without the context added by wrapping errors; for other errors the status is `codes.Internal` with `err`'s text (`apierror.go:FromError`).

## Side Effects
- None. Pure error construction and conversion.

## Verification
- Run: `go test ./internal/apierror/...`
