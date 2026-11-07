# testserver

## Overview

Command `testserver` is the one binary behind every plain helper pod the egress e2e suites stand up. A cobra subcommand picks its behavior, so a single ko image backs all of them: `grpc` (cleartext HTTP/2 echo origin), `http` (HTTP/1.1 origin), `egressprobe` (egress-gateway driver), `websocket`, `kill`, and `procstatus`. The `.yaml.tmpl` files in this directory deploy the subcommands as fixtures.

## Interface

- `main.go:main` - registers the six subcommands `newGRPCCmd`, `newHTTPCmd`, `newEgressProbeCmd`, `newKillCmd`, `newProcStatusCmd`, `newWebsocketCmd`.
- `grpc.go:newGRPCCmd` - serves `grpcechopb` `Echo`/`EchoStream`/`EchoBidi` (`grpc.go:echoServer`) plus the gRPC health service on `--listen` (default `:50051`), and an HTTP `/readyz` on `--health-listen`.
- `http.go:newHTTPCmd` - `/healthz` (optional `--body`), `/echoheaders` (`http.go:echoedHeaders`), `POST /fetch`, `/dial` (`http.go:dial`); `--accept-connect` wraps it in `http.go:acceptingConnect`.
- `egressprobe.go:newEgressProbeCmd` - `/handshake?sni=&credential-bundle=` returning `egressprobe.go:handshakeResult` (`OK`, `Stage`, `ConnectStatus`, `ChainPEM`); flags `--gateway-address`, `--trust-bundle`, `--handshake-timeout`.
- `websocket.go:newWebsocketCmd` - `/ws` answering PING with PONG or, with `--echo`, echoing every message; `/readyz`.
- `kill.go:newKillCmd` / `kill.go:killMatching(procDir, contains, self)` - SIGKILLs matching processes under `/proc`.
- `procstatus.go:newProcStatusCmd` / `procstatus.go:readProcesses` - prints `internal/e2e/procstatus` JSON for PID 1 and `--name` matches.
- `tls.go:serveHTTP` / `tls.go:configureTLSServer` - cleartext plus optional HTTPS on the fixed `tls.go:tlsListenAddress` (`:8443`, TLS 1.2+, HTTP/1.1 only).

## Invariants

- Each pod runs exactly one subcommand, so a pod's wire behavior is unchanged from separate binaries: the grpc pod speaks nothing but cleartext HTTP/2 (`main.go` package doc).
- The grpc origin serves no TLS because the leg under test is the tunnel, and the egress gateway relays a terminated CONNECT as opaque TCP (`grpc.go:newGRPCCmd`).
- The grpc health listener binds before `Serve`, so a readiness 200 can never precede the gRPC port being open; it is off by default because an h2c mux on the RPC listener would defeat the egress fixture (`grpc.go:newGRPCCmd`).
- `EchoStream` requires a positive count of at most 1000 (`grpc.go:maxStreamCount`); `EchoBidi` answers each request as it arrives so frames move both ways at once (`grpc.go:echoServer.EchoBidi`).
- The egress probe's stage constants name the failing hop: `client`, `gateway_tls`, `connect`, `tunnel`, `inner_handshake`; `dialStage` reads atunnel's typed errors (`egress.ErrGatewayHandshake`, `egress.ConnectRejectedError`), not message prose (`egressprobe.go` stage consts, `egressprobe.go:dialStage`).
- The egress probe requires `credential-bundle` per request: the credential is the variable under test and nothing is cached across handshakes (`egressprobe.go:handshake`).
- The inner handshake uses `InsecureSkipVerify` deliberately: verification of the minted chain is the caller's assertion (`egressprobe.go:fetchChain`).
- `--tls-cert` and `--tls-key` must be provided together, and invalid credentials fail setup before listening (`tls.go:configureTLSServer`); both listeners are bound before either serves (`tls.go:serveListeners`).
- `kill` fails when nothing matched, and ESRCH counts as killed because a gofer or virtiofsd can exit on its own (`kill.go:newKillCmd`, `kill.go:killMatching`).
- Process names come from argv[0], not `/proc/<pid>/comm`, because runsc re-executes `/proc/self/exe` and comm truncates cloud-hypervisor (`procstatus.go:processName`); a missing status field is an error (`procstatus.go:statusFields.number`).

## Side Effects

- Binds TCP listeners (defaults `:50051`, `:8080`, `:80`; HTTPS on `:8443`) (`grpc.go`, `http.go`, `websocket.go`, `tls.go`).
- Opens outbound mTLS CONNECT tunnels to the egress gateway via `egress.NewClient` and completes inner TLS handshakes (`egressprobe.go:fetchChain`).
- Sends SIGKILL to processes in its PID namespace (`kill.go:killMatching`).
- Reads `/proc` (cmdline, status, task/*/status) (`kill.go`, `procstatus.go`).

## Verification

- Run: `go test ./internal/e2e/fixtures/testserver/...`
