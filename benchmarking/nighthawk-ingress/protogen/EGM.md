# protogen

Builds `nighthawk.desc`, the Nighthawk `FileDescriptorSet` that `spec.py` and
`output.py` load at runtime. Keeping the descriptor out of the repository
avoids vendoring nighthawk, envoy, xds, protoc-gen-validate, and googleapis
proto trees.

## Interface

- `generate_descriptor.sh` - compiles `api/adaptive_load/*.proto`, `api/client/*.proto`, and `api/request_source/*.proto` from a nighthawk checkout into a self-contained descriptor set (`--include_imports`) at the output path (default `/out/nighthawk.desc`, or `$1`).

## Invariants

- Expects the source trees pre-cloned by the Dockerfile protogen stage at `/src/nighthawk`, `/src/envoy/api`, `/src/xds`, `/src/pgv`, `/src/googleapis`, with nighthawk and envoy at their pinned commits (`generate_descriptor.sh` header comment).
- The `Dockerfile` pins `NIGHTHAWK_COMMIT`, `ENVOY_COMMIT`, and `NIGHTHAWK_IMAGE_DIGEST` together: the digest is the Docker Hub tag for that nighthawk commit, and the envoy commit is nighthawk's pin (`benchmarking/nighthawk-ingress/Dockerfile` ARG block).

## Side Effects

- Writes the descriptor set file to `$OUT` and prints its byte count (`generate_descriptor.sh`).

## Verification

- ["None"] (exercised only inside the Docker image build; no test invokes it)
