# Phase 1: Critical Security Improvements

## Overview

- **Rate limiting** on login attempts — ✅ **DONE**, implemented differently than originally scoped here (see below)
- **Account lockout** after failed attempts — ✅ **DONE**, same mechanism as rate limiting
- **Audit logging** for authentication events — ⏳ **PENDING**
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

## 3. Audit Logging — ⏳ PENDING

### Database Migration (`0001_XXX_audit_log.sql`)
```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    event_type TEXT NOT NULL,
    user_id UUID REFERENCES users(id),
    email TEXT,
    ip_address TEXT,
    user_agent TEXT,
    metadata JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```
Migration numbers must be chosen against the actual current sequence in `authservice/migrations/` at implementation time — it has moved past `0001_011` since this doc was written.

### Event Types
- `auth.login.success/failure`
- `auth.logout`
- `auth.refresh.success/failure`
- `auth.register`
- `auth.verify_email`
- `auth.password_change/reset`
- `auth.account_locked/unlocked`
- `auth.service_client.auth`
- `auth.admin.create/change_account`

### Implementation
- New package: `authservice/internal/audit/`
- Async logging option for performance
- 90-day retention with daily cleanup routine

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

## Environment Variables (pending items only)

```bash
# Audit
AUDIT_RETENTION_DAYS=90
AUDIT_ASYNC=true

# Encryption
ENCRYPTION_MASTER_KEY=<base64-256-bit>
```
