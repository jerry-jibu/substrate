# pki

Writes the throwaway certificates the benchmark processes share, standing in
for the pod-identity and service certificates a cluster projects. One CA signs
everything; the fake atelet keeps the CA key to sign actor certificates.

## Interface

- `pki.go` file-name constants - `CAFile` (`ca.pem`), `CAKeyFile` (`ca-key.pem`), `WorkerBundle` (`worker.pem`), `AteletBundle` (`atelet.pem`), `RouterBundle` (`router.pem`), `GatewayBundle` (`gateway.pem`), all relative to the output directory.
- `pki.go:Options` - `WorkerHosts` and `GatewayHosts` for the serving certificates' names and IPs, and `Validity`.
- `pki.go:Generate` - creates the CA and the four leaf bundles (worker, atelet, router, gateway) in `dir`, creating it if needed.
- `pki.go:LoadCA` - reads back the CA certificate and signing key that `Generate` wrote.
- `pki.go:PEMCertificate` / `pki.go:Serial` - PEM-encode a DER certificate; random serial number.

## Invariants

- `Generate` requires nonempty `WorkerHosts` and `GatewayHosts`; an empty `Validity` defaults to 30 days (`pki.go:Generate`).
- Worker and atelet leaf certificates carry the same `substratex509.PodIdentity` node (`bench-node`, a fixed UID), because atunnel checks that atelet's certificate names the same node as its own (`pki.go:node`, `pki.go:Generate`).
- The router bundle carries the router SPIFFE ID and client-auth EKU; the gateway bundle carries server-auth EKU for its hosts (`pki.go:Generate` leaves table).
- Leaf bundles are written leaf, then CA, then key: the layout of a Kubernetes Pod Certificate credential bundle (`pki.go:Generate`).
- `LoadCA` fails if the PEM holds no certificate, no key, or a key that cannot sign (`pki.go:LoadCA`).
- Keys are ECDSA P-256; the CA key file is written mode 0600, certificates 0644 (`pki.go:Generate`).

## Side Effects

- `Generate` creates `dir` and writes six files into it (`pki.go:Generate`).
- `LoadCA` reads `ca.pem` and `ca-key.pem` from `dir` (`pki.go:LoadCA`).

## Verification

- Run: `go test ./benchmarking/atunnel/internal/pki/...` from the repo root.
