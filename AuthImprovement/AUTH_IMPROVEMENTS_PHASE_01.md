# Phase 1: Critical Security Improvements

## Overview

- **Rate limiting** on login attempts — ✅ **DONE**, implemented differently than originally scoped here (see below)
- **Account lockout** after failed attempts — ✅ **DONE**, same mechanism as rate limiting
- **Audit logging** for authentication events — ✅ **DONE**, implemented differently than originally scoped here (see below)
- **JWT private key encryption** at rest — ✅ **DONE**, implemented differently than originally scoped here (see below)

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

## 4. JWT Private Key Encryption — ✅ DONE (boolean discriminator + key ring, not the 3-column sketch)

Shipped with deliberate deviations from the original scoping above:

- **Schema differs from the original 3-column sketch.** Instead of separate
  `encryption_iv`/`encryption_tag` columns, `jwt_keys.private_key` is reused
  directly: for encrypted rows it holds `base64(nonce || ciphertext || tag)`,
  exactly what Go's `cipher.AEAD.Seal` produces in one call — no separate
  IV/tag marshaling needed. Two columns were added instead of three:
  `private_key_encrypted BOOLEAN NOT NULL DEFAULT FALSE` (explicit
  discriminator; existing rows default to `FALSE`, which is what makes
  "gradual migration, old keys stay readable" free with no backfill
  statement) and `encryption_key_id TEXT` (nullable — a **fingerprint** of
  the master key, `SHA-256(masterKey)` truncated, never the key itself).
  Migration: `authservice/migrations/0001_013_encrypt_jwt_private_key.sql`.
- **`swlib/encryption/`** (`encryption.go`, `keyring.go`) ships `Encrypt`/
  `Decrypt`/`ParseMasterKey`/`Fingerprint` plus a `KeyRing` type — see master
  key rotation below. `authservice/internal/db/jwt_keys.go` gained
  `encodeForStorage`/`decodeFromStorage` helpers used by `createNewKeyPair`
  (encrypts before `INSERT`) and `GetSigningKey` (decrypts after `SELECT`,
  passing through unchanged when `private_key_encrypted = FALSE`).
- **No forced backfill.** The row that was plaintext at deploy time stays
  plaintext-readable until it naturally rotates out (`keysNeedRotation`
  already runs hourly, replacing keys ~3 days before their 30-day expiry —
  the replacement is encrypted automatically since `createNewKeyPair` always
  encrypts going forward). An operator runbook (bump `valid_until` to force
  early rotation) covers accelerating this after deploy.
- **Master key rotation, not in the original doc at all.** `ENCRYPTION_MASTER_KEY`
  is the required "current" key (mandatory, fail-fast at startup if unset/
  invalid — a silent plaintext fallback would defeat the feature). An
  optional `ENCRYPTION_MASTER_KEY_PREVIOUS` (comma-separated) holds retired
  keys, used only to decrypt rows still encrypted under an older key.
  `swlib/encryption.KeyRing` looks up the right key by a row's
  `encryption_key_id` fingerprint. Rotation is restart-activated (config is
  read at startup only) and bounded: a retired key only needs to stay
  configured until every currently-valid row has rotated onto the new one
  (~30-day key lifetime), confirmed via `SELECT DISTINCT encryption_key_id
  FROM jwt_keys WHERE valid_until > now();`, after which it's dropped.
- **Expired `jwt_keys` rows are now cleaned up, which they never were
  before.** `cleanupExpiredJwtKeys` (`internal/db/jwt_keys.go`), wired into
  the existing hourly `DoDatabaseMaintenance` alongside `cleanupAuditLog`/
  `cleanupSecurityThrottle`, deletes rows expired more than
  `JWT_KEY_RETENTION_DAYS` (default 7) ago — a small forensics/clock-skew
  margin, not the key's full 30-day lifetime. Safe by construction: both
  `GetSigningKey` and `GetVerificationKeys` already filter
  `valid_until > now()`, so a row past retention was already unreachable;
  deletion only reclaims storage and shrinks how much historical encrypted
  key material sits in the database.
- A decrypt failure (wrong/rotated master key, corrupted data) fails only
  the specific signing call — the service, health checks, and verification
  of already-issued tokens keep running.

---

## Environment Variables

```bash
# Audit (shipped)
AUDIT_RETENTION_DAYS=90    # default 90
AUDIT_BUFFER_SIZE=1000     # default 1000; async writer channel buffer

# Encryption (shipped)
ENCRYPTION_MASTER_KEY=<base64-256-bit>            # required; openssl rand -base64 32
ENCRYPTION_MASTER_KEY_PREVIOUS=<key1>,<key2>,...  # optional; retired keys, for rotation
JWT_KEY_RETENTION_DAYS=7                          # default 7; expired jwt_keys row cleanup
```
