# internal

## Overview

Packages internal to the `benchmark-egress` binary: the subcommand
implementations and the target certificate issuer.

## Subdirectories & Boundaries

- cmd: [./cmd/EGM.md](./cmd/EGM.md) - the `actor` and `target` subcommands of the egress scale test.
- targetcert: [./targetcert/EGM.md](./targetcert/EGM.md) - name-constrained CA and wildcard certificate for the target's HTTPS.
