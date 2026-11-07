# cmd

## Overview

Subcommands of the `benchmark-egress` binary: the two sides of the egress
scale test's data path. `actor` holds one tunnel per endpoint open through
the gateway and measures it; `target` answers the requests behind the
Services.

## Subdirectories & Boundaries

- actor: [./actor/EGM.md](./actor/EGM.md) - the workload: control routes on port 80, per-endpoint request loops, Stats with four distributions.
- target: [./target/EGM.md](./target/EGM.md) - the endpoint: fixed-body responses over HTTP and HTTPS on one or many ports.
