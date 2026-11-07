# sandbox/client

## Overview

REPL client for the stateless sandbox demo. It connects to ateapi, resumes
the sandbox actor, forwards each typed command to the actor's `/process`
endpoint through the local atenet proxy, and suspends the actor again on
exit (`client/main.go:79`-`:104`).

## Interface

- Flags (`client/main.go:51`-`:56`): `--name` and `--atespace` (both
  required), `--ateapi` (defaults to port-forwarding `svc/api`), `--atenet`
  (default `localhost:8000`), `--kubeconfig`, `--context`.
- `main` (`client/main.go:50`): dials ateapi with
  `ateclient.NewClient`, calls `ResumeActor` (`client/main.go:88`), runs the
  REPL loop, and defers `SuspendActor` (`client/main.go:95`-`:104`).
- `runCommand` (`client/main.go:165`): POSTs
  `{"command": ["sh", "-c", line]}` to `http://<atenet>/process` with the
  `atenet.TargetActorHeader` routing header set to
  `atespace/name`.
- REPL commands: any shell line runs remotely; `exit` or EOF leaves
  (`client/main.go:141`-`:143`).

## Invariants

- `--name` and `--atespace` are required and the client exits fatally
  without them (`client/main.go:59`-`:64`).
- The actor is always suspended on exit, even after an interrupt
  (`client/main.go:67`-`:77`, `:95`-`:104`).
- Commands run as `sh -c <line>` inside the actor; output and errors are
  printed as received (`client/main.go:146`-`:160`).

## Side Effects

- Mutates cluster state: resumes the actor at startup and suspends it at
  exit through the ateapi Control service (`client/main.go:88`, `:98`).
- Sends each command to the atenet proxy, which routes it to the actor
  (`client/main.go:165`-`:184`).

## Verification

- Run: `go test ./demos/...`
