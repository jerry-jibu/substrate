# ateapi/internal/store/dockerenv

## Overview

Points testcontainers at the active Docker context. It lives apart from
storetest because storetest imports atepg, so atepg cannot import storetest
back, and both need this before starting a container (`dockerenv.go` package
doc).

## Interface

- `Configure` (`dockerenv.go`) - set the environment testcontainers reads (`DOCKER_HOST`, `TESTCONTAINERS_DOCKER_SOCKET_OVERRIDE`) from the Docker CLI context, unless already set.
- `Required` (`dockerenv.go`) - whether an unavailable container must fail the test rather than skip it: true when `CI=true` or `REQUIRE_DOCKER=true`.

## Invariants

- `Required` is the named fail-closed predicate for CI: a skip and a pass are the same exit code, so CI must fail (`dockerenv.go` `Required`).
- On darwin the reaper socket override is `/var/run/docker.sock` because the reaper runs inside the Docker VM (`dockerenv.go` `Configure`).

## Side Effects

- Runs `docker context inspect` and mutates the process environment (`dockerenv.go` `Configure`).

## Verification

- Run: `go test ./cmd/ateapi/internal/store/dockerenv/...`
