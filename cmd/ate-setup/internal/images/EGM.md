# ate-setup/internal/images

## Overview

Decides where ate-setup gets container images from. The manifests name images
as `ko://` import paths, which ko builds and publishes, or as Dockerfile
contexts built with docker buildx; a pre-built install instead rewrites the
references onto an already-published image set (`images.go` package doc).

## Interface

- `ModulePath` (`images.go`) - the repository module path a `ko://` reference is joined to.
- `Components`, `ReleaseComponents`, `IsReleased`, `KoReference`, `ImageName` (`images.go`) - the closed list of publishable component packages and the `ko://` reference and image name conventions.
- `DockerfileImage`, `EnvoyDataplane`, `ReleaseDockerfileImages` (`images.go`) - images built from Dockerfiles rather than ko.
- `Source` / `IsPrebuilt` / `Describe` (`images.go`) - where images come from: build from source with ko, or pull a published set.
- `Prebuilt`, `NewPrebuilt` (`prebuilt.go`) - the drop-in counterpart to `ko.Runner` that rewrites `ko://` references to published images, pinning each tag to a digest via a `Digester`.
- `BuildDockerfileImage`, `PublishDockerfileImage` (`dockerfile.go`) - docker buildx build/push of context-path images.
- `SourceLabel`, `SourceURL`, `RevisionLabel`, `VersionLabel`, `CheckReleaseVersion`, `CheckReleaseTag`, `StaleTags`, `Platforms`, `Incomplete`, `CheckPublished` (`release.go`) - release-tag validation, build platforms, and post-push completeness checks.
- `Digester`, `RemoteDigest` (`digest.go`) - resolve a tag to its current digest with one HEAD request per image.

## Invariants

- The component list is closed on purpose: a `ko://` reference not in `Components` names something the installer cannot deploy (`images.go` `Components`).
- Prebuilt rewriting matches the whole reference, which is what makes it safe; substitution is textual and covers CRD fields as well as pod specs (`prebuilt.go`).
- The pinned digest is not decoration: an ActorTemplate's container image and other images must be digest-pinned (`prebuilt.go` `Pin`).
- `CheckReleaseVersion` / `CheckReleaseTag` accept only release tags of the form vMAJOR.MINOR.PATCH[-PRERELEASE] (`release.go`).

## Side Effects

- `BuildDockerfileImage` / `PublishDockerfileImage` run `docker buildx` and push to the registry (`dockerfile.go`).
- `RemoteDigest` performs a registry HEAD request per image (`digest.go`).
- `CheckPublished` reads the registry to find missing per-platform images (`release.go`).

## Verification

- Run: `go test ./cmd/ate-setup/internal/images/...`
