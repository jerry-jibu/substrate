# postgressetup

## Overview
The PostgreSQL identity setup shared by Substrate installers. `setup.go` embeds `setup.sql` and names the schema, roles, and users it creates; the SQL is a parameterized psql script driven by `Config.PSQLArgs`.

## Interface
- `Config` naming `Schema`, `OwnerRole`, `OwnerUser`, `OwnerPassword`, `ReadWriteRole`, `ReadWriteUser`, `ReadWritePassword`; `DefaultConfig()` returns the passwordless bundled identities (`setup.go:Config`, `setup.go:DefaultConfig`)
- `Config.PSQLArgs()` returns the `--set=substrate_*` psql arguments the SQL reads (`setup.go:PSQLArgs`)
- `SQL()` returns the parameterized script; `Script()` wraps it in `BEGIN`/`COMMIT` (`setup.go:SQL`, `setup.go:Script`)
- Default identity constants: `Schema = "substrate"`, `OwnerRole = "substrate_owner"`, `OwnerUser = "substrate_owner_user"`, `ReadWriteRole = "substrate_readwrite"`, `ReadWriteUser = "substrate_readwrite_user"` (`setup.go` const block)

## Invariants
- The script takes a `pg_advisory_xact_lock` on `agent-substrate:setup:<schema>`, so concurrent setups serialize (`setup.sql` DO block).
- A managed role whose attributes differ from the required ones (login flag, superuser, createdb, createrole, replication, or inherit on a login user) raises an exception instead of being altered (`setup.sql` role attribute check).
- An existing schema owned by someone other than `owner_role` raises an exception (`setup.sql` schema ownership check).
- `CREATE` on schema `public` is revoked from PUBLIC, and the managed schema grants only USAGE to the readwrite role, with default privileges granting it SELECT/INSERT/UPDATE/DELETE on tables, USAGE/SELECT/UPDATE on sequences, EXECUTE on routines, and USAGE on types (`setup.sql` grant/revoke block).
- The caller of `SQL()` supplies `Config.PSQLArgs` and owns the surrounding transaction (`setup.go:SQL` doc comment).

## Side Effects
- When executed, the script creates or alters login roles, sets their passwords (`ALTER ROLE ... PASSWORD`), grants role membership, creates the schema if absent, and revokes public access (`setup.sql` DO block).

## Verification
- Run: `go test ./pkg/postgressetup/...` from the repo root.
