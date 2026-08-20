# Phase 1: Critical Security Improvements

## Overview

- **Rate limiting** on login attempts — ✅ **DONE**, implemented differently than originally scoped here (see below)
- **Account lockout** after failed attempts — ✅ **DONE**, same mechanism as rate limiting
- **Audit logging** for authentication events — ✅ **DONE**, implemented differently than originally scoped here (see below)
- **JWT private key encryption** at rest — ⏳ **PENDING**

---

## 1. Rate Limiting — ✅ DONE (Postgres, not Redis)

This doc originally proposed a Redis-backed sliding-window limiter. What actually shipped is a Postgres-backed one instead — no new infra dependency, same effect:

- `security_throttle` table (migration `0001_011_create_security_throttle_table.sql`) tracks attempts per scope (e.g. login, `ScopeEmailSendByIP`) with a sliding window.
- `authservice/internal/db/throttle.go` — `RecordAttemptResult`, `IsAttemptLocked`.
- `authservice/internal/server/throttle.go` — `ThrottleConfig`, wired into login/token endpoints in `authentication.go` and `cmd/authservice/main.go`.
- Separately, general per-request gRPC rate limiting (in-memory token bucket) exists at `swlib/ratelimit/limiter.go` + `swlib/http/middlewares/ratelimit.go`, applied via `app.RateLimitInterceptor`.

No further work needed here unless requirements change (e.g. multi-instance deployment where a shared Redis-backed limiter would matter more than it does today).

---

## 2. Account Lockout — ✅ DONE (same table as rate limiting)

Progressive lockout is implemented via the same `security_throttle` sliding-window mechanism above, rather than the separate `login_attempts` / `account_lockouts` tables originally proposed. No further work needed.

---

## 3. Audit Logging — ✅ DONE (`internal/db`/`internal/server`, not a standalone package)

Shipped with three deliberate deviations from the original scoping above:

- **No standalone `internal/audit/` package.** Follows the same
  writer-on-`*DB` + thin-wrapper-on-`*AuthServer` split already used for
  `security_throttle` (`internal/db/throttle.go` + `internal/server/throttle.go`):
  `internal/db/audit.go` (`AuditEvent`, `InsertAuditEvent`, `cleanupAuditLog`)
  and `internal/server/audit.go` (`AuditWriter`, per-event-type emit helpers).
- **Real async writer**, not fire-and-forget-synchronous: `AuditWriter` wraps
  a buffered channel (size via `AUDIT_BUFFER_SIZE`, default 1000); handlers
  call a non-blocking `emit` that drops (and logs) on a full buffer rather
  than blocking the request. `cmd/authservice/main.go`'s `auditFlusher`
  background routine drains the channel to the database, with a bounded
  drain of any remaining buffered events on shutdown.
- **Retention cleanup folded into the existing hourly `dbMaintenance`**
  routine (`cleanupAuditLog`, alongside `cleanupSecurityThrottle`) rather
  than a separate daily ticker — `AUDIT_RETENTION_DAYS` (default 90) is
  threaded through `DoDatabaseMaintenance`.
- **`auth.account_unlocked` was dropped from the event list.** Lockout is
  purely a `locked_until` timestamp that expires on its own; there's no code
  path that ever executes "unlock," so emitting a synthetic unlock event
  would misrepresent what happened. `auth.account_locked` fires at the real
  transition point instead — see `RecordAttemptResult`'s `locked` return
  value in `internal/db/throttle.go`, consumed by
  `internal/server/throttle.go`'s `recordLoginAttempt`/`recordClientAttempt`.

Migration: `authservice/migrations/0001_012_create_audit_log_table.sql`
(`audit_log` table + indexes on `created_at`, `event_type`, `user_id`).

### Event Types (as shipped)
- `auth.login.success` / `auth.login.failure`
- `auth.logout`
- `auth.refresh.success` / `auth.refresh.failure`
- `auth.register`
- `auth.verify_email`
- `auth.password_change` / `auth.password_reset`
- `auth.account_locked`
- `auth.service_client.auth`
- `auth.admin.create` / `auth.admin.change_account`

---

## 4. JWT Private Key Encryption — ⏳ PENDING

### Approach: AES-256-GCM with env var master key

### Database Migration
```sql
ALTER TABLE jwt_keys ADD COLUMN encryption_key_id TEXT;
ALTER TABLE jwt_keys ADD COLUMN encryption_iv TEXT;
ALTER TABLE jwt_keys ADD COLUMN encryption_tag TEXT;
```

### Implementation
- New package: `swlib/encryption/`
- Master key: `ENCRYPTION_MASTER_KEY` (base64-encoded 256-bit)
- Generate key: `openssl rand -base64 32`
- Encrypt before storing, decrypt in-memory only for signing
- Supports gradual migration (existing keys remain readable)
- Touches `authservice/internal/db/jwt_keys.go`, where private keys are currently read/written in plaintext

---

## Environment Variables

```bash
# Audit (shipped)
AUDIT_RETENTION_DAYS=90    # default 90
AUDIT_BUFFER_SIZE=1000     # default 1000; async writer channel buffer

# Encryption (pending)
ENCRYPTION_MASTER_KEY=<base64-256-bit>
```
