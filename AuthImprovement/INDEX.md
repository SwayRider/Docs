# Index - Auth Improvement Plan

Security hardening plan for `authservice`. `AUTH_IMPROVEMENTS.md` is the full analysis; the phase files below hold implementation detail for what's still pending. See `authservice/CODE_REVIEW_2026-08.md` for the code-verified audit trail this plan has been reconciled against.

| File | Title | Status |
| --- | --- | --- |
| [AUTH_IMPROVEMENTS](./AUTH_IMPROVEMENTS.md) | Security Analysis & Vulnerability Summary | Living doc |
| [AUTH_IMPROVEMENTS_PHASE_01](./AUTH_IMPROVEMENTS_PHASE_01.md) | Critical Security | Done — rate limiting, lockout, audit logging & JWT key encryption at rest all done |
| [AUTH_IMPROVEMENTS_PHASE_02](./AUTH_IMPROVEMENTS_PHASE_02.md) | High Impact | Partially Done — cookie `Secure`/`SameSite`, HIBP breach detection and password history done; TLS deferred (to do later) |
| [AUTH_IMPROVEMENTS_PHASE_03](./AUTH_IMPROVEMENTS_PHASE_03.md) | Enhanced Security | Partially Done — user enumeration protection and MFA (all 5 plans) done; sessions & secrets management pending |
| [multifactor/](./multifactor/INDEX.md) | MFA implementation plans | Implemented — all 5 plans complete (TOTP core+db → authservice endpoints → gateway/client → Flutter data+login → Flutter UI), 2026-08-21 |
| [AUTH_IMPROVEMENTS_PHASE_04](./AUTH_IMPROVEMENTS_PHASE_04.md) | Operational | Pending — metrics, key rotation audit trail, service-client hardening |

## Already implemented (not originally scoped this way)

- Login rate limiting & progressive account lockout — Postgres `security_throttle` table, not the Redis design originally proposed in Phase 1.
- Audit logging for authentication events — async writer (`internal/db/audit.go` + `internal/server/audit.go`), not the standalone `internal/audit/` package originally proposed in Phase 1.
- RS256 JWT signing with automatic, advisory-lock-guarded key rotation, 3072-bit RSA; the private key is AES-256-GCM-encrypted at rest (`swlib/encryption`), keyed by `ENCRYPTION_MASTER_KEY` with `ENCRYPTION_MASTER_KEY_PREVIOUS`-based rotation support — a boolean discriminator + key fingerprint column, not the 3-column IV/tag design originally proposed in Phase 1.
- Single-use, SHA-256-hashed refresh tokens with atomic consume; IP/user-agent used only as a soft signal.
- Cookie `Secure` flag auto-derived from `X-Forwarded-Proto`.
- Cookie `SameSite` configurable via `COOKIE_SAMESITE` (default `strict`).
- User enumeration protection (uniform responses), fixed 2026-08-17.
- Password breach detection against the Pwned Passwords API (k-anonymity range protocol, fail-open), fixed 2026-08-20.
- Password history enforcement (recent-password reuse prevention, `password_history` table, fail-open), fixed 2026-08-20.
- Audit events for breached-password and reused-password rejections (`auth.password_breached_rejected`, `auth.password_reuse_rejected`), fixed 2026-08-20.
