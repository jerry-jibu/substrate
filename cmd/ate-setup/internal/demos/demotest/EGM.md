# ate-setup/internal/demos/demotest

## Overview

Shared test fixtures for the demo packages: an Env that needs no cluster, and
assertions that a rendered template is a complete manifest (`demotest.go`
package doc).

## Interface

- `Env(t)` (`demotest.go`) - builds a `steps.Env` with no cluster connection: resolves settings from the process environment (so tests can steer a demo's settings with `t.Setenv`) against `config.RepoRoot()`.
- `AssertRendered(t, manifest)` (`demotest.go`) - no `${PLACEHOLDER}` remains on a non-comment line and the result decodes as a Kubernetes manifest stream via `kube.DecodeManifestBytes`.
- `AssertRenderedActorTemplate(t, manifest, want)` (`demotest.go`) - the manifest strictly parses into the `ateapi.ActorTemplate` it will be created as, names the given `resources.ActorTemplateRef`, and leaves `ko://` references as written.

## Invariants

- `AssertRenderedActorTemplate` fails if the rendered manifest sets `snapshotConfig.storageLocation`, because the API rejects it without its preview gate (`demotest.go`).
- Comments are exempt from the placeholder check; some document placeholders rather than substitute them (`demotest.go` `assertNoPlaceholders`).

## Side Effects

- ["None"] (helpers read the process environment and repository files only).

## Verification

- Run: `go test ./cmd/ate-setup/internal/demos/demotest/...`
