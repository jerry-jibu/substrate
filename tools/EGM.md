# tools

Standalone Go dev/CI tools. Root-module tools run as `go run ./tools/<name>`;
tools with their own `go.mod` (`apitool`, `junittool`) run as
`go -C tools/<name> run .` and pin their dependencies separately from the
main module (`hack/site.sh`, `.github/workflows/pr-workflow.yaml`).

## Subdirectories & Boundaries

- [./apitool/EGM.md](./apitool/EGM.md) - validates `api/ateapi/v1/ateapi.proto` against the API style guidelines; own go.mod.
- [./junittool/EGM.md](./junittool/EGM.md) - records the JUnit files a CI run will write; own dependency-free go.mod so it runs on CI's critical path.
- [./site-content/EGM.md](./site-content/EGM.md) - generates Hugo content stubs in `site/content/` from `docs/` and demo READMEs.
- [./site-release-data/EGM.md](./site-release-data/EGM.md) - writes the release metadata the website templates read.
- [./validate-image-cache/EGM.md](./validate-image-cache/EGM.md) - batch-validates OCI images against `internal/imagecache` pull/parse/unpack.

## Global Invariants

- Each tool directory keeps its own `go.mod` when it must not share the root module's dependency graph (`tools/junittool/EGM.md`, AGENTS.md repository layout).

## System Verification

- Root-module tools: `go test ./tools/...` from the repo root. Tools with their own `go.mod`: `cd tools/<name> && go test ./...`.
