# probe

## Overview

Command `probe` is a minimal introspection actor used by the e2e suites: it reports what the runtime looks like from inside the actor, so tests can assert on real in-actor state rather than the config atelet generates. Serves an HTTP mux on `:80`. The templates under this directory (`probe.yaml.tmpl` and variants: sized, parking, multiactor, bootfetch, nonroot) deploy it as a Substrate fixture. The `nonroot` variant is its own binary in `./nonroot`.

## Interface

- `main.go:main` - serves `/whoami`, `/fetch`, `/readfile`, `/writefile`, `/lookup`, `/stat`, `/resources`, `/capabilities`, `/filesystem`, `/healthz`; supports the `filesystem-child` subcommand; when `PROBE_BOOT_FETCH_URL` is set, gates `/healthz` on the restore-fetch logic.
- `main.go:capabilities` - capability sets from `/proc/self/status` decoded by `main.go:decodeCapMask` and `main.go:capabilityNames` (bits without a name report as `CAP_<n>`).
- `main.go:whoami` - identity from the bind-mounted `/run/ate` files (`main.go:identityFile` etc.) plus `heldIdentity`, an fd opened at startup and read at offset 0 via `main.go:readAllAt`.
- `main.go:fetch` - egress-path HTTP(S) GET with `roots=bundle|system|pem`, repeatable `header=<name>:<value>`, and `dial=<host:port>` overrides; reports status, first 64 KiB of body, and `tls_handshake` (`complete`/`closed`/`failed`).
- `main.go:bootFetch` / `main.go:restoreFetch` - boot-time and restore-time gated fetches behind `main.go:bootFetchURLEnv`.
- `filesystem.go:filesystem` - POST creates non-root-owned files; GET reports their metadata; `?nonroot=1` and `?icmp=1` run the deeper checks.
- Subdirectory: [nonroot](./nonroot/EGM.md) - the non-root-image probe binary.

## Invariants

- Capabilities are read from `/proc/self/status`, not the OCI spec, so the report proves the sandbox actually applied the requested set; `capabilityNames` order is fixed by the kernel and must not be sorted (`main.go:procStatus`, `main.go:capabilityNames`).
- `fetch` fails closed on unknown `roots` values and on a `ca` parameter that does not pair with `roots=pem`, so a suite typo cannot flip a negative control (`main.go:fetch`).
- `fetch` does not follow redirects: a cross-scheme redirect would hop between the gateway's cleartext and TLS legs and flip the credential-injection behavior under test (`main.go:fetch`).
- A `dial` address must parse as host:port; the URL's host still supplies SNI and Host (`main.go:fetch`).
- `bootFetch` does not retry: the actor's EgressPolicy exists before resume, so a failure is the finding; a probe that cannot boot-fetch exits and never answers its wakeup probe (`main.go:bootFetch`, `main.go:bootFetchURLEnv`).
- `heldIdentity` is deliberately held open from before serving, so snapshots carry a live guest file handle that a restore must re-bind; `whoami` reads through it every request (`main.go:heldIdentity`).
- `restoreFetch.healthz` answers 503 until the actor named in the identity file has fetched the URL, fetching at most once per identity in the background (`main.go:restoreFetch`).
- The filesystem check requires UID 1000 to create and read its private file, UID 2000 to be denied, device-node creation to fail with EPERM, and (with `?icmp=1`) a non-root ICMP echo socket to open (`filesystem.go:checkNonRootFilesystem`, `filesystem.go:checkUnprivilegedICMP`).
- Files created by the filesystem check are chowned to 1000:1000 with mode 0755 plus setgid, at the paths in `filesystem.go:filesystemPaths`, and must survive checkpoint/restore (`filesystem.go:initializeFilesystemFile`).

## Side Effects

- Binds a listener on port 80 (`main.go:main`).
- Reads `/proc/self/status`, `/proc/meminfo`, `/sys/fs/cgroup/cpu.max`, `/sys/fs/cgroup/memory.max`, and the `/run/ate` projections (`main.go`, `main.go:resources`).
- Issues outbound HTTP(S) requests through the actor's egress path (`main.go:fetch`, `main.go:bootFetch`).
- Writes and chowns files under `/var/worker-security` and `/home/counter/worker-security`; re-executes itself as other UIDs via `syscall.Credential` and attempts `mknod` (expected EPERM) (`filesystem.go`).

## Verification

- Run: `go test ./internal/e2e/fixtures/probe/...`
