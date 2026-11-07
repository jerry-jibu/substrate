# update

Scripts that rewrite the current tree: formatting, generated code, module
vendor lists, and license inventories. Each has a corresponding entry in
`../update-all.sh` (see `README.md`), and a verify counterpart under
`../verify/`.

## Interface

- `gofmt.sh` - runs `$(go env GOROOT)/bin/gofmt -s -w` on every directory with tracked or untracked `.go` files, excluding `vendor/`, `third_party/`, and `_LICENSES/`.
- `proto-fmt.sh` - runs `clang-format` (must be on PATH) over tracked `.proto` files without reflowing long lines.
- `codegen.sh` - `go generate ./...`, then protobuf Go code via `hack/protoc.sh` with `protoc-gen-go`/`protoc-gen-go-grpc` from `hack/run-tool.sh`, from the repo root so files register under their repo paths.
- `go-modules.sh` - `go mod vendor` then `go mod tidy`.
- `licenses.sh` - rebuilds the `_LICENSES/` directory with `go-licenses` for linux/darwin on amd64/arm64, working around the tool's module-replacement handling with a temporary `go.work`.

## Invariants

- Every script here must have an entry in `../update-all.sh` (`README.md`).
- `gofmt.sh` uses the GOROOT gofmt, not whatever is on PATH, because `GOTOOLCHAIN` does not affect gofmt's version (`gofmt.sh` comment).
- The codegen steps run in order because some tools depend on others' output (`codegen.sh` comment).
- `proto-fmt.sh` does not reflow long lines, since reflowed comments would require re-running the proto generators and make `update-all` awkward (`proto-fmt.sh` comment).
- Protobuf code is compiled from the repo root with `paths=source_relative` so each file registers under its repo path (`codegen.sh:codegen::protobuf`).

## Side Effects

- Rewrites working-tree files: Go formatting, proto formatting, generated Go, `vendor/` and `go.mod`/`go.sum`, and deletes then rebuilds `_LICENSES/` (`licenses.sh` `rm -rf "${OUTDIR}"`).

## Verification

- Run: `hack/verify/shellcheck.sh` (shellchecks every repo `.sh` script); the corresponding `../verify/*.sh` scripts check the same trees without rewriting.
