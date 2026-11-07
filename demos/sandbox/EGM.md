# sandbox

## Overview

Stateless sandbox demo: an actor that runs one command per HTTP request and
returns its output, plus a REPL client that resumes the actor, forwards
lines to it, and suspends it when done. Install templates provision the
WorkerPool the sandbox actor runs on.

## Interface

- `main` (`main.go:42`): HTTP server on `PORT` (default 80) exposing POST
  `/process` (`main.go:44`-`:53`).
- `handleProcess` (`main.go:56`): decodes
  `ProcessRequest{command, envvars, cwd, timeout}` (`main.go:28`), runs the
  command with `exec.CommandContext`, and returns
  `ProcessResponse{stdout, stderr, exitCode, error}` (`main.go:35`).
- client: [./client/EGM.md](./client/EGM.md) - the REPL driving the sandbox
  actor through atenet.
- Install templates: `sandbox.yaml.tmpl` (WorkerPool and namespace) and
  `sandbox-template.yaml.tmpl` (the ActorTemplate); `manual-test-multi.yaml`
  exercises multiple actors.

## Invariants

- A request with an empty `command` array is rejected with 400
  (`main.go:63`-`:66`).
- A `timeout` string is parsed with `time.ParseDuration` and bounds the
  command's context; invalid values are rejected with 400 (`main.go:69`-`:78`).
- A command that fails without an `ExitError` (for example, not found)
  reports exit code -1 (`main.go:106`-`:111`).

## Side Effects

- Executes arbitrary commands inside the actor sandbox with caller-supplied
  env vars and working directory (`main.go:80`-`:97`).
- The client resumes and suspends a real actor around each session
  (`client/main.go:88`, `:98`).

## Verification

- Run: `go test ./demos/...`
