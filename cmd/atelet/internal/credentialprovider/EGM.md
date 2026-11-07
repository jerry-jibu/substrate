# credentialprovider

## Overview

Resolves image registry credentials for atelet by invoking kubelet image
credential provider plugins (the kubelet CredentialProviderConfig/executable
contract). Reusing the kubelet contract keeps atelet free of any cloud SDK;
the plugin and its config reach atelet as read-only host mounts (see
`manifests/ate-install/atelet.yaml`). Exposes a go-containerregistry
`authn.Keychain`.

## Interface

- `credentialprovider.go:New` - loads the CredentialProviderConfig at `configPath`, resolving each provider's executable inside `binDir`; returns a `*Keychain`.
- `credentialprovider.go:Keychain` - implements `authn.Keychain` and `authn.ContextKeychain`; safe for concurrent use.
- `credentialprovider.go:ResolveContext` - returns the first candidate authenticator, or `authn.Anonymous` if none.
- `credentialprovider.go:Candidates` - returns credentials of all providers matching the target, most specific auth key first, then config order.
- `config.go:loadConfig` (unexported) - reads and validates the kubelet config; errors only on the config as a whole, skips unusable providers.
- `config.go:newPlugin` (unexported) - validates one provider entry and binds it to its executable.
- `match.go:matchesImage` (unexported) - kubelet `URLsMatch` semantics: equal host-segment count, `filepath.Match` per segment, equal ports, glob path is a prefix.
- `match.go:matchingAuthKeys`, `match.go:sortAuthKeys`, `match.go:registryOf` (unexported) - auth-key matching, reverse-lexical ordering, registry extraction.
- `exec.go:(*plugin).exec` (unexported) - runs the plugin binary once: `CredentialProviderRequest` JSON on stdin, `CredentialProviderResponse` JSON on stdout.
- `testdata/*.sh` - fake provider executables used by tests.

## Invariants

- Provider names must be bare file names; a name containing a path separator is rejected so it cannot escape `binDir` (`config.go:newPlugin`).
- Only `credentialproviderv1.SchemeGroupVersion` is spoken; providers configured or responding at another apiVersion are rejected rather than misread (`config.go:supportedAPIVersion`, `exec.go:(*plugin).exec`).
- `tokenAttributes` providers are refused: actors have no Kubernetes service account to mint a TokenRequest for (`config.go:newPlugin`).
- A zero or negative cache duration is raised to `minCacheDuration` (1 minute) (`credentialprovider.go:(*plugin).store`); a provider naming a positive but shorter lifetime may hold a genuinely short-lived credential and is not stretched.
- Responses with an unrecognized `cacheKeyType` are not cached at all (`credentialprovider.go:(*plugin).store`).
- An empty auth map is cached too, so "no credentials" costs no further subprocess (`credentialprovider.go:cacheEntry`).
- One plugin invocation is bounded by `execTimeout` (1 minute) and its stdout by `maxResponseBytes` (1 MiB) (`exec.go:execTimeout`, `exec.go:maxResponseBytes`).
- Failing providers are logged and skipped, like the kubelet; an empty keychain makes pulls anonymous (`credentialprovider.go:Candidates`, `config.go:loadConfig`).

## Side Effects

- Executes provider binaries found under `binDir` as subprocesses, passing the process env unioned with the provider's configured env (`exec.go:(*plugin).exec`).
- Reads the credential provider config file from disk (`config.go:loadConfig`).
- Holds per-plugin in-memory response caches guarded by a mutex, swept on every store (`credentialprovider.go:plugin`).

## Verification

- Run: `go test ./cmd/atelet/internal/credentialprovider/...`
