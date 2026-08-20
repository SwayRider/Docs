# Auth Improvements & Security Analysis

## Executive Summary

Security analysis of `authservice`, tracking what's implemented against a phased improvement plan. This doc predates a 2026-08 audit pass (`authservice/CODE_REVIEW_2026-08.md`) that independently found and fixed several of the same gaps — that file is the current, code-verified source of truth for recent fixes; this doc has been updated to match it and to correct stale figures, but treat `CODE_REVIEW_2026-08.md` as authoritative where the two disagree.

**Status at a glance**: rate limiting, account lockout, JWT rotation, single-use hashed refresh tokens, and user-enumeration protection are implemented (though not always via the mechanism originally proposed here — see notes per section). Audit logging, JWT-key-at-rest encryption, TLS enforcement, breach-password detection, password history, MFA, session management, secrets management, metrics, and service-client secret rotation are still pending — see `AuthImprovement/INDEX.md` for the phase-by-phase status table.

---

## Current Architecture

### Components
| Component | Port | Purpose |
|-----------|------|---------|
| gRPC Service | 8081 | Internal service-to-service communication |
| REST API | 8080 | HTTP API via grpc-gateway |
| Web Server | 8000 | Email verification/reset pages |

### Security Features
- RS256 JWT signing with automatic, advisory-lock-guarded key rotation
- Argon2id password hashing (64 MiB memory, 3 iterations, 4 threads)
- Single-use, SHA-256-hashed refresh tokens with atomic consume; IP/user-agent recorded as a soft signal, never a gate
- Postgres-backed login rate limiting and progressive account lockout (`security_throttle` table)
- General gRPC rate limiting (in-memory token bucket, `swlib/ratelimit`)
- Service client authentication with OAuth2 client credentials
- Endpoint-level security profiles (public, unverified, admin, service client)
- Password entropy validation (minimum 80 bits)
- Uniform responses across login/registration to prevent user enumeration

---

## Security Analysis

### 1. Password Security

**Strengths**
- Argon2id hashing: winner of the Password Hashing Competition, resistant to GPU and side-channel attacks
- Parameters: 64 MiB memory, 3 iterations, 4 threads (`swlib/crypto/hashing.go`)
- Constant-time comparison via `subtle.ConstantTimeCompare`
- Password entropy validation: minimum 80 bits required

**Still pending**
- No password history — users can reuse previous passwords (Phase 2)
- No password expiration policy
- No breached-password detection (Phase 2, HaveIBeenPwned)

---

### 2. JWT Token Security

**Strengths**
- RS256 signing, asymmetric, public keys for verification
- Automatic key rotation with advisory locks to prevent race conditions (`authservice/internal/db/jwt_keys.go`)
- 3072-bit RSA (`swlib/crypto/keypair.go`) — above the 2048-bit minimum
- 30-day key validity, regular rotation limits exposure window

**Still pending**
- Private keys stored in plaintext in PostgreSQL — no encryption at rest (Phase 1)
- No usage tracking or rotation audit trail (Phase 4)
- Single active signing key at a time (old keys remain for verification only)

---

### 3. Refresh Token Security

**Strengths**
- Single-use tokens: atomically consumed and replaced (`authservice/internal/db/refresh_tokens.go`)
- SHA-256-hashed storage
- 64-byte random tokens, cryptographically secure generation
- 30-day expiration
- IP/user-agent are recorded and used only as a soft signal — never a gate, since NAT/load-balanced clients legitimately share IPs and user agents are trivially spoofed

**Still pending**
- No token revocation list beyond single-use rotation

---

### 4. Session Management

**Strengths**
- HttpOnly cookies — prevents JavaScript access to refresh tokens
- Configurable `SameSite` (default `Strict`) — CSRF protection
- `Secure` flag now auto-derived from `X-Forwarded-Proto` (`authentication.go`, `CookieHeaderMatcher`) rather than defaulting to false
- Cookie namespacing prevents collisions

**Still pending**
- No session timeout beyond refresh-token expiration (30 days) (Phase 3)
- No concurrent session control or session listing/revocation (Phase 3)

---

### 5. Authentication Flow

**Strengths**
- Separate public/verified/admin endpoint profiles
- Service client authentication via a distinct OAuth2 client-credentials flow, with scoped permissions
- **Rate limiting and account lockout are implemented** — a Postgres `security_throttle` table with a sliding window tracks attempts per scope (login, email-send-by-IP, etc.) and applies progressive lockout; see `authservice/internal/db/throttle.go` and `authservice/internal/server/throttle.go`. This was originally scoped as a Phase 1 Redis-backed design (see `AUTH_IMPROVEMENTS_PHASE_01.md`) but shipped against Postgres instead — no new infra dependency required.

**Still pending**
- No MFA support — single-factor authentication only (Phase 3)
- No login anomaly detection

---

### 6. Email Verification

**Strengths**
- Token-based verification: 64-byte secure random tokens, time-limited, single-use
- Email send rate limiting per source IP (`ScopeEmailSendByIP` in the throttle system)

**Still pending**
- Token appears in URL — visible in logs/browser history
- No email deliverability validation, only format validation

---

### 7. Password Reset

**Strengths**
- Token-based, separate from verification tokens; secure random generation; time-limited
- Covered by the same rate-limiting/throttle system as login

**Fixed since this doc was first written**
- User enumeration protection: responses are now uniform for valid/invalid emails (fixed 2026-08-17, see `authservice/CODE_REVIEW_2026-08.md`)

**Still pending**
- Token appears in URL — visible in logs

---

### 8. Service Client Security

**Strengths**
- Standard OAuth2 client-credentials flow
- Scope-based access control
- Hashed secrets (Argon2id)
- Admin-only creation

**Still pending**
- No secret rotation policy — secrets never expire (Phase 4)
- No audit logging of service client usage (Phase 1/4)
- No IP restrictions per service client

---

### 9. Infrastructure Security

**Strengths**
- PostgreSQL advisory locks prevent race conditions during key rotation
- Standard `sql.DB` connection pooling
- Graceful shutdown

**Still pending**
- Database connections default to `sslmode=disable` (`authservice/internal/db/postgres.go`) (Phase 2)
- No gRPC TLS enforcement (Phase 2)
- No secrets manager — configuration via plain environment variables (Phase 3)
- Health endpoints publicly accessible

---

### 10. Logging & Monitoring

**Strengths**
- Structured logging with function/component context
- Failed login attempts logged
- No sensitive data (passwords/hashes) in logs

**Still pending**
- No persistent audit trail of auth events (Phase 1)
- No alerting on suspicious activity (Phase 4)
- No Prometheus metrics (Phase 4)

---

## Vulnerability Summary

| Category | Severity | Issue | Status |
|----------|----------|-------|--------|
| JWT | HIGH | Private keys stored unencrypted in database | Pending (Phase 1) |
| Monitoring | HIGH | No audit logging | Pending (Phase 1) |
| Password | MEDIUM | No breached password detection | Pending (Phase 2) |
| Password | MEDIUM | No password history | Pending (Phase 2) |
| ~~Session~~ | ~~MEDIUM~~ | ~~`SameSite` not configurable to `Strict`~~ | **Fixed** — configurable via `COOKIE_SAMESITE`, default `Strict` |
| Infrastructure | MEDIUM | Database can run without TLS | Pending (Phase 2) |
| Service Clients | MEDIUM | No secret rotation policy | Pending (Phase 4) |
| Monitoring | MEDIUM | No anomaly detection | Pending |
| ~~Authentication~~ | ~~HIGH~~ | ~~No rate limiting on login attempts~~ | **Fixed** — Postgres `security_throttle` |
| ~~Authentication~~ | ~~HIGH~~ | ~~No account lockout after failed attempts~~ | **Fixed** — same mechanism |
| ~~Session~~ | ~~MEDIUM~~ | ~~Secure flag defaults to false~~ | **Fixed** — derived from `X-Forwarded-Proto` |
| ~~Auth flow~~ | ~~MEDIUM~~ | ~~User enumeration via inconsistent responses~~ | **Fixed** 2026-08-17 |

---

## Implementation Roadmap

See `AuthImprovement/INDEX.md` for the current phase-by-phase status. Phase docs (`AUTH_IMPROVEMENTS_PHASE_01.md` … `_04.md`) retain full design detail only for items still pending; implemented items are trimmed to a status note and code pointer.

---

## Appendix: Relevant Code Locations

| Component | File |
|-----------|------|
| Password hashing | `swlib/crypto/hashing.go` |
| JWT key generation & rotation | `authservice/internal/db/jwt_keys.go`, `swlib/crypto/keypair.go` |
| Refresh tokens | `authservice/internal/db/refresh_tokens.go`, `authservice/internal/model/refresh_token.go` |
| Login rate limiting / lockout | `authservice/internal/db/throttle.go`, `authservice/internal/server/throttle.go` |
| General gRPC rate limiting | `swlib/ratelimit/limiter.go`, `swlib/http/middlewares/ratelimit.go` |
| Cookie handling | `swlib/http/cookies/cookie.go` |
| Database connection / TLS | `authservice/internal/db/postgres.go` |
| Login flow | `authservice/internal/server/authentication.go` |
| Prior audit findings | `authservice/CODE_REVIEW_2026-08.md` |
