# ateapi/internal/store/atepg/migrations

## Overview

Goose SQL migrations defining the ateapi PostgreSQL schema: Substrate resource
tables plus the OpenFGA storage schema, kept in the same schema and migration
directory so resource mutations and authorization tuple updates execute within
one transaction (`000001_initial.sql`).

## Interface

- `000001_initial.sql` (goose `Up` migration) - creates the schema:
  - Substrate tables: `atespaces`, `actors`, `actor_egress_policies`, `actor_templates`, `tags`, `workers`, `worker_assignments` (with `worker_assignments_worker_idx`), `worker_outbox` (partitioned, with `worker_outbox_default` UNLOGGED partition and `worker_outbox_xid`), `worker_outbox_trim`, `leases` (with `leases_expires_at_idx`), `global_access_policy`, `atespace_access_policies`.
  - OpenFGA tables (pinned to OpenFGA PostgreSQL migration version 6): `tuple`, `authorization_model`, `store`, `assertion`, `changelog`.

## Invariants

- Rows carry the full resource proto in a `proto bytea` column keyed by (atespace, name) or name; `actors`, `actor_templates`, and `tags` reference `atespaces(name)` with `ON DELETE RESTRICT` (`000001_initial.sql`).
- `actor_egress_policies` cascades from `actors(atespace, name)` (`000001_initial.sql`).
- Workers are global-scoped and named by pod UID, so `name` alone is the primary key and `uid` is unique (`000001_initial.sql` `workers`).
- `worker_outbox` ordering is by `xid` (exactly one row per transaction), retention by `created_at` partitions, and it is UNLOGGED for durability tradeoffs; `worker_outbox_trim` stays LOGGED to preserve the high-water mark across restarts (`000001_initial.sql` comments).
- `worker_outbox.created_at` must use `clock_timestamp()` rather than `now()`, so slow transactions do not route into expired partitions (`000001_initial.sql`).
- `global_access_policy` is a singleton: its `id` column only admits `true` (`000001_initial.sql`).

## Side Effects

- Executed by the migration runner in `../atepg` against the target database.

## Verification

- Run: `go test ./cmd/ateapi/internal/store/atepg/...`
