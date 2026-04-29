# Phase 3: Enhanced Security Improvements

## Overview

Phase 3 adds advanced security features:
- **Multi-Factor Authentication (MFA)** - TOTP-based, no external provider needed
- **Session management** - Listing, idle timeout, concurrent limits
- **User enumeration protection** - Consistent responses for all auth flows
- **Secrets management** - Encrypted file for production, plain env vars for dev

**Timeline**: 4 weeks
**Priority**: MEDIUM-HIGH

---

## 1. Multi-Factor Authentication (TOTP)

### How TOTP Works

TOTP is an open standard (RFC 6238) that works **completely in-house**:
1. Server generates a secret key (20 bytes)
2. Secret shared with user via QR code
3. User adds to authenticator app (Google Authenticator, Authy)
4. App generates 6-digit codes every 30 seconds using HMAC-SHA1
5. Server validates codes using the same algorithm

**No external service, no API calls, no cost.**

### Configuration
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| MFA Enabled | `MFA_ENABLED` | true |
| Code Length | `MFA_CODE_LENGTH` | 6 |
| Time Step | `MFA_TIME_STEP` | 30 seconds |
| Grace Period | `MFA_GRACE_PERIOD` | 1 (accept prev/next code) |
| Backup Codes | `MFA_BACKUP_CODES` | 10 |

### New Package: `backend/swlib/totp/`

```go
// Package totp implements TOTP (RFC 6238) for multi-factor authentication.

type Config struct {
    SecretSize  int           // Bytes for secret generation (default: 20)
    CodeLength  int           // Digits in code (default: 6)
    TimeStep    time.Duration // Seconds per code (default: 30)
    GracePeriod int           // Accept adjacent codes (default: 1)
}

func GenerateSecret() (string, error)
func GenerateCode(secret string, t time.Time, cfg Config) (string, error)
func Validate(secret, code string, t time.Time, cfg Config) (bool, error)
func GenerateQRCodeURL(secret, email, issuer string) string
```

### Database Migration (`0001_012_mfa.sql`)
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
- 10 single-use codes during MFA setup
- 8-character alphanumeric codes
- Stored as Argon2id hashes
- Can regenerate (invalidates old codes)

---

## 2. Session Management

### Features
1. **Session listing** - Users can see active sessions
2. **Session revocation** - Users can terminate specific sessions
3. **Idle timeout** - Auto-logout after inactivity (default: 15 min)
4. **Concurrent session limit** - Max 5 active sessions per user

### Configuration
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Idle Timeout | `SESSION_IDLE_TIMEOUT` | 900 seconds |
| Max Sessions | `SESSION_MAX_CONCURRENT` | 5 |

### Database Migration (`0001_013_sessions.sql`)
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
- On login: Check concurrent limit, revoke oldest if exceeded
- On refresh: Check idle timeout, update last_activity
- Session stored with device info for user recognition

---

## 3. User Enumeration Protection

### Changes
| Endpoint | Current | Improved |
|----------|---------|----------|
| Login | Generic message ✓ | - |
| Password Reset | Async ✓ | Add random delay |
| Registration | Shows "email exists" | Generic "check your email" |
| Email Verification | Async ✓ | - |

### Implementation
- Add random delays (0-1000ms) to async operations
- Return consistent messages for all flows
- Minimum processing time (500ms) for registration

---

## 4. Secrets Management

### Design
- **Development**: Unencrypted env vars / `.env` files
- **Production**: Encrypted secrets file, decrypted with master key

### Configuration
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Environment | `ENV` | development |
| Secrets File | `SECRETS_FILE` | (empty) |
| Master Key | `SECRETS_MASTER_KEY` | (empty) |

### New Package: `backend/swlib/secrets/`

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
# Generate master key
secrets-tool generate-key

# Encrypt secrets file
secrets-tool encrypt secrets.json plaintext.json

# Decrypt secrets file
secrets-tool decrypt plaintext.json secrets.json
```

---

## File Structure

```
backend/
├── swlib/
│   ├── totp/
│   │   ├── totp.go
│   │   ├── qr.go
│   │   └── totp_test.go
│   └── secrets/
│       ├── secrets.go
│       └── secrets_test.go
├── services/authservice/
│   ├── cmd/
│   │   ├── authservice/main.go (modified)
│   │   └── secrets-tool/main.go (new)
│   ├── internal/
│   │   ├── db/
│   │   │   ├── mfa.go (new)
│   │   │   └── sessions.go (new)
│   │   ├── model/
│   │   │   ├── mfa.go (new)
│   │   │   └── session.go (new)
│   │   └── server/
│   │       ├── mfa.go (new)
│   │       ├── sessions.go (new)
│   │       ├── login.go (modified)
│   │       ├── registration.go (modified)
│   │       └── password_reset.go (modified)
│   └── migrations/
│       ├── 0001_012_mfa.sql (new)
│       └── 0001_013_sessions.sql (new)
└── protos/auth/v1/auth.proto (modified)
```

---

## Environment Variables

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

## Rollout Plan

### Week 5-6: MFA
- TOTP library and tests
- MFA endpoints
- Backup codes
- Login flow integration

### Week 7: Session Management
- Session CRUD
- Idle timeout
- Concurrent limits

### Week 8: Enumeration & Secrets
- User enumeration protection
- Secrets management
- Integration testing

---

## Testing Strategy

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
