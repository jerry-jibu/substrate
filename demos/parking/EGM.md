# parking

## Overview

Request-parking demo: more actors than workers, driven through a
request-then-suspend loop, showing that atenet-router's parked-request
buffer turns transient saturation into slower 200s instead of 503s. There is
no workload binary; the demo is install templates plus a load generator.

## Interface

- `load.sh` (`load.sh:17`): oversubscription load generator. Options `-d`
  duration seconds (default 30), `-a` atespace (default
  `ate-demo-parking`), and positional actor IDs (default `p1 p2 p3 p4`)
  (`load.sh:43`-`:77`).
- Preflight (`load.sh:101`-`:105`): creates the atespace and actors
  idempotently with `kubectl ate`, template `parking`.
- Load loop (`load.sh:136`-`:146`): per actor, runs a `kubectl ate proxy
  actor` local proxy (ports from 18001, `load.sh:111`), curls it recording
  `%{http_code} %{time_total}`, and suspends the actor after each request.
- Result tally (`load.sh:157`-`:194`): counts 200/503/other and reports
  average and slowest 200 latency; exits nonzero when no request succeeded
  or a non-200/503 status appeared.
- `parking.yaml.tmpl` and `parking-template.yaml.tmpl`: the WorkerPool (2
  workers) and the parking ActorTemplate the script targets
  (`load.sh:96`, `:47`).

## Invariants

- Each proxy is waited on for its `Proxying` ready line before load starts,
  and a proxy that fails to start aborts the run (`load.sh:114`-`:133`).
- The script cleans up child proxies and its temp directory on EXIT, INT,
  and TERM (`load.sh:82`-`:92`).
- With parking ON the expected outcome is ~all 200s; parking OFF
  (`--parked-request-max=0` on the router) yields a burst of 503s
  (`load.sh:26`-`:29`).

## Side Effects

- Creates atespaces and actors, starts local port proxies, and suspends
  actors in the target cluster via `kubectl ate` (`load.sh:102`-`:144`).
- Writes per-actor logs under a `mktemp -d` directory removed on exit
  (`load.sh:79`-`:91`).

## Verification

- No automated test; `load.sh` itself is the check and requires a running
  cluster with the demo installed (`load.sh:38`-`:42`).
