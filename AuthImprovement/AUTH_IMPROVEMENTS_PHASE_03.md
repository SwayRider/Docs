# Phase 3: Enhanced Security Improvements

## Overview

- **Multi-Factor Authentication (MFA)** — ✅ **DONE** (all five implementation plans complete: TOTP core + db, authservice endpoints, gateway + authclient, Flutter data + login, Flutter UI)
- **Session management** — ⏳ **PENDING**
- **User enumeration protection** — ✅ **DONE** (fixed 2026-08-17, see below)
- **Secrets management** — ⏳ **PENDING**

---

## 1. Multi-Factor Authentication (TOTP) — ✅ DONE (all five implementation plans complete, 2026-08-21)

> **Status:** implemented across five sequential plans under [`Docs/AuthImprovement/multifactor/`](multifactor/INDEX.md) (TOTP core + db → authservice endpoints → gateway/client → Flutter data + login → Flutter UI) — server, gateway, and app all landed; the Flutter app can enroll (manual key + QR), log in with TOTP or a backup code, and disable/regenerate from the profile. Key decisions that deviate from or refine the sketch below: **manual base32 key is the primary enrollment path** (a phone cannot scan its own screen, so QR is secondary, server-rendered, for second-device enrollment); the **MFA challenge token is DB-backed** (SHA-256-hashed, TTL + attempt counter — mirrors reset tokens) rather than a bare short-lived token; the **TOTP secret is encrypted at rest** with `ENCRYPTION_MASTER_KEY` (like the JWT key) instead of plain `TEXT`; a **new `mfa` throttle scope** bounds TOTP guessing (login lockout alone doesn't — a phishing attacker can loop successful password logins); `MFA_ENABLED=false` fails closed on management endpoints and bypasses the login step.

### How TOTP Works

TOTP is an open standard (RFC 6238) that works **completely in-house**:
1. Server generates a secret key (20 bytes)
2. Secret shared with user via QR code
3. User adds to authenticator app (Google Authenticator, Authy)
4. App generates 6-digit codes every 30 seconds using HMAC-SHA1
5. Server validates codes using the same algorithm

No external service, no API calls, no cost.

### Configuration (target)
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| MFA Enabled | `MFA_ENABLED` | true |
| Code Length | `MFA_CODE_LENGTH` | 6 |
| Time Step | `MFA_TIME_STEP` | 30 seconds |
| Grace Period | `MFA_GRACE_PERIOD` | 1 (accept prev/next code) |
| Backup Codes | `MFA_BACKUP_CODES` | 10 |

### New Package: `swlib/totp/`

```go
type Config struct {
    SecretSize  int
    CodeLength  int
    TimeStep    time.Duration
    GracePeriod int
}

func GenerateSecret() (string, error)
func GenerateCode(secret string, t time.Time, cfg Config) (string, error)
func Validate(secret, code string, t time.Time, cfg Config) (bool, error)
func GenerateQRCodeURL(secret, email, issuer string) string
```

### Database Migration
```sql
CREATE TABLE user_mfa (
    id SERIAL PRIMARY KEY,
    user_id UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    enabled BOOLEAN NOT NULL DEFAULT false,
    secret TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE mfa_backup_codes (
    id SERIAL PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash TEXT NOT NULL,
    used BOOLEAN NOT NULL DEFAULT false,
    used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```
Migration number must be chosen against the actual current sequence in `authservice/migrations/` at implementation time.

### New gRPC Endpoints
- `SetupMFA` - Start enrollment (returns secret/QR URL)
- `EnableMFA` - Verify code and enable MFA
- `DisableMFA` - Disable MFA (requires password)
- `GetMFAStatus` - Check if MFA is enabled
- `VerifyMFA` - Verify TOTP code during login
- `GenerateBackupCodes` - Generate new backup codes

### Modified Login Flow
```
User logs in → Password valid → MFA enabled?
  ├─ No  → Return tokens
  └─ Yes → Return MFA token (short-lived)
           User submits TOTP code → Verify → Return tokens
```

### Backup Codes
- 10 single-use codes during MFA setup, 8-character alphanumeric
- Stored as Argon2id hashes
- Can regenerate (invalidates old codes)

---

## 2. Session Management — ⏳ PENDING

### Features
1. Session listing — users can see active sessions
2. Session revocation — users can terminate specific sessions
3. Idle timeout — auto-logout after inactivity (default: 15 min)
4. Concurrent session limit — max 5 active sessions per user

### Configuration (target)
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Idle Timeout | `SESSION_IDLE_TIMEOUT` | 900 seconds |
| Max Sessions | `SESSION_MAX_CONCURRENT` | 5 |

### Database Migration
```sql
CREATE TABLE user_sessions (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    refresh_token_hash TEXT NOT NULL,
    ip_address TEXT NOT NULL,
    user_agent TEXT,
    device_name TEXT,
    last_activity TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL
);
```

### New gRPC Endpoints
- `ListSessions` - Get active sessions
- `RevokeSession` - Terminate specific session
- `RevokeAllSessions` - Terminate all except current

### Behavior
- On login: check concurrent limit, revoke oldest if exceeded
- On refresh: check idle timeout, update `last_activity`
- Session stored with device info for user recognition

---

## 3. User Enumeration Protection — ✅ DONE

Fixed 2026-08-17 (see `authservice/CODE_REVIEW_2026-08.md`): login, registration, and password-reset flows now return uniform responses regardless of whether the email/account exists. No further work needed here; the original plan's "add random delay" and "minimum processing time" refinements remain optional hardening, not required.

**2026-08-24 exception:** invite-only registration's rejection (`Register` in invite-only mode, non-invited email) deliberately reverted to a distinct `codes.PermissionDenied` rather than the uniform response — an owner-approved tradeoff so the mobile app can show an "invitation required" message, accepted because the invite pool is small, invited users register quickly, and completing registration for an invited email requires mailbox access regardless of whether invite status is known. Scoped to this one branch only: duplicate-email uniform-response protection in `Register`, and the `InviteUser`/`GetToken` fixes, are unaffected. See `CLAUDE.md` and `authservice/review/CODE_REVIEW_2026-08.md` finding #10.

---

## 4. Secrets Management — ⏳ PENDING

### Design
- **Development**: unencrypted env vars / `.env` files
- **Production**: encrypted secrets file, decrypted with a master key

### Configuration (target)
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Environment | `ENV` | development |
| Secrets File | `SECRETS_FILE` | (empty) |
| Master Key | `SECRETS_MASTER_KEY` | (empty) |

### New Package: `swlib/secrets/`

```go
type Manager struct {
    env         string
    secretsFile string
    encryptor   *encryption.AESGCMEncryptor
    cache       map[string]string
}

func (m *Manager) Get(key string) string {
    // Check cache (production) or env var (development)
}
```

### CLI Tool (`cmd/secrets-tool/`)
```bash
secrets-tool generate-key
secrets-tool encrypt secrets.json plaintext.json
secrets-tool decrypt plaintext.json secrets.json
```

---

## Environment Variables (pending items only)

```bash
# MFA
MFA_ENABLED=true
MFA_CODE_LENGTH=6
MFA_TIME_STEP=30
MFA_GRACE_PERIOD=1
MFA_BACKUP_CODES=10

# Session Management
SESSION_IDLE_TIMEOUT=900
SESSION_MAX_CONCURRENT=5

# Secrets Management
ENV=development
SECRETS_FILE=
SECRETS_MASTER_KEY=
```

---

## Testing Strategy (pending items only)

### Unit Tests
- TOTP code generation and validation
- Backup code generation and verification
- Session CRUD operations
- Secrets encryption/decryption

### Integration Tests
- MFA setup and verification flow
- Login with MFA enabled
- Session listing and revocation
- Idle timeout enforcement
- Concurrent session limits

### Security Tests
- TOTP code reuse prevention
- Backup code single-use enforcement
- Timing attack resistance
- Session hijacking prevention
