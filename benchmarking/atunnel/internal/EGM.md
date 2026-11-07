# atunnel/internal

Shared packages for the atunnel benchmark, each standing in for one piece of a
real worker or its peers.

## Subdirectories & Boundaries

- [fakeatelet](./fakeatelet/EGM.md) - serves the WorkerSupport calls atunnel makes (certificate minting, egress policy) on the node-local socket, with atelet's identity.
- [load](./load/EGM.md) - the load generator: open-loop pacer, traffic modes, scenarios, and the rung ladder that reads atunnel's CPU and memory per rate.
- [pki](./pki/EGM.md) - writes and reads back the throwaway CA and credential bundles all processes share.
- [procstats](./procstats/EGM.md) - process resource snapshots (CPU, RSS, heap, goroutines, cgroup v2) and the `/stats` HTTP handler that serves them.
- [report](./report/EGM.md) - data types for what a run measured, the table renderer, and the JSON writer.
- [sink](./sink/EGM.md) - the fake peers: the ingress actor per sandbox and the mTLS egress gateway with Envoy's HTTP/2 settings.
- [workerhost](./workerhost/EGM.md) - hosts atunnel with the worker's own code, making `cmd/ateworker-gvisor`'s calls per actor; Linux only.
