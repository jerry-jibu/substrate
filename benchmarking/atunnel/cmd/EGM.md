# atunnel/cmd

Go command tree for the atunnel benchmark. Holds the single binary,
`atunnel-bench`, whose subcommands are the benchmark's processes.

## Subdirectories & Boundaries

- [atunnel-bench](./atunnel-bench/EGM.md) - the `atunnel-bench` binary: subcommands `pki`, `sink`, `load`, and the Linux-only `tunnel`, `worker`, `ingress-actor`, `egress-actor`, `netns-exec`.
