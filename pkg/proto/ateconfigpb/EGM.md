# proto/ateconfigpb

## Overview
Protobuf definition and generated Go bindings for the unified `ate-api-server` configuration object. `apiconfig.proto` is the source of truth; `apiconfig.pb.go` is `protoc-gen-go` output (`DO NOT EDIT`, protoc-gen-go v1.36.11-devel).

## Interface
- `APIConfig` message with a `logging` field, accessor `GetLogging()` (`apiconfig.proto:APIConfig`, `apiconfig.pb.go:APIConfig`)
- `APILogging` message with a `level` string (`debug`, `info`, `warn`, or `error` per the proto comment), accessor `GetLevel()` (`apiconfig.proto:APILogging`, `apiconfig.pb.go:APILogging`)
- Go package path `go.ate.dev/substrate/pkg/proto/ateconfigpb` (`apiconfig.proto:go_package`)

## Invariants
- The Go bindings are generated; edit `apiconfig.proto` and regenerate rather than editing `apiconfig.pb.go` (`apiconfig.pb.go:15` header).

## Side Effects
- None.

## Verification
- Run: `go test ./pkg/proto/ateconfigpb/...` from the repo root.
