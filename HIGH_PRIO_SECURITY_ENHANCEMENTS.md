# High-Priority Security Enhancements

Cross-cutting security work that should be done soon. Service-specific hardening
lives in [AuthImprovement/INDEX.md](./AuthImprovement/INDEX.md).

## 1. Give authservice its own database user (not the Postgres superuser)

**Status:** open

**Problem.** `authservice` connects to the shared Postgres as the superuser
(`DB_USER=${POSTGRES_USER}` in `infra/dev/layer-20/compose.yml`, `postgresadmin`
in the example env). The other services follow the same pattern and share the
same account. Any compromised service can read, change or drop every other
service's database, including `authdb` with the users, refresh tokens and the
(encrypted) JWT signing keys.

**Fix.**
- Create one login role per service that owns only that service's database
  (`CREATE ROLE <svc> LOGIN PASSWORD '…'; CREATE DATABASE <svc>db OWNER <svc>;`),
  with `CONNECT` revoked from `PUBLIC` on the other databases.
- Point each service's `DB_USER` / `DB_PASSWORD` at its own role and keep the
  superuser for administration only (the infra layer, backups, provisioning).
- `authservice` currently creates its database at startup (`EnsureDatabase`),
  which needs superuser rights. Move that to an idempotent provisioning step in
  the infra repo (role + database), and make `EnsureDatabase` optional or remove
  it. Migrations keep running at startup as the owning role.
- Store each password as its own secret; rotate them when the superuser
  password is rotated.

**Order.** `authservice` first (most sensitive data), then the other services.
New services should be built this way from the start (own role and database,
created by the infra step).
