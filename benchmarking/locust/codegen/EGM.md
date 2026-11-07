# codegen

Generates the Python gRPC clients the locust tests import into
`benchmarking/locust/common`. The generated files are not checked in:
protoc's Python output puts the whole serialized descriptor on one line, so
concurrent proto changes would always conflict on it.

## Interface

- `generate.sh:python_proto` - compiles one `.proto` into `OUT_DIR` (default `benchmarking/locust/common`) with `grpc_tools.protoc`, emitting `<base>_pb2.py` and `<base>_pb2_grpc.py`.
- `generate.sh` default proto set - with no arguments, compiles `api/ateapi/v1/ateapi.proto` and `internal/proto/glutton/glutton.proto`.
- `generate.sh` environment - `OUT_DIR` redirects the output directory; `PYTHON` supplies an interpreter that already has `requirements.txt` installed (as the Dockerfiles do), skipping the venv.

## Invariants

- `python_proto` rewrites the grpc file's protoc-emitted `import <base>_pb2` to a relative `from . import` so it resolves under the `common` package (`generate.sh:python_proto` sed step).
- The rewrite goes through a temp file because `sed -i` differs between GNU and BSD sed (`generate.sh:python_proto`).

## Side Effects

- Writes `*_pb2.py` / `*_pb2_grpc.py` files into `benchmarking/locust/common` (or `OUT_DIR`) and creates a venv at `benchmarking/locust/codegen/venv` when `PYTHON` is unset (`generate.sh`).

## Verification

- Run: `hack/verify/python-protos.sh` (runs this script in CI; see its header comment).
