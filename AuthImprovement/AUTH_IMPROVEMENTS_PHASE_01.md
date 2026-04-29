# Phase 1: Critical Security Improvements

## Overview

Phase 1 addresses the most critical security gaps in the authservice:
- **Rate limiting** on login attempts (Redis)
- **Account lockout** after failed attempts
- **Audit logging** for authentication events
- **JWT private key encryption** at rest

**Timeline**: 2 weeks
**Priority**: CRITICAL

---

## 1. Rate Limiting (Redis)

### Configuration
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Redis Host | `REDIS_HOST` | localhost |
| Redis Port | `REDIS_PORT` | 6379 |
| Redis Password | `REDIS_PASSWORD` | (empty) |
| IP Rate Limit | `RATE_LIMIT_IP_MAX` | 10 attempts per 60s |
| Email Rate Limit | `RATE_LIMIT_EMAIL_MAX` | 5 attempts per 300s |

### Implementation
- New package: `backend/swlib/ratelimit/`
- Uses sliding window counter pattern in Redis
- Key format: `ratelimit:ip:{ip}` and `ratelimit:email:{email}`
- Returns `codes.ResourceExhausted` when limited
- Resets counters on successful login

### Docker Compose
Add Redis container to `infra/dev/layer-00/compose.yaml`

---

## 2. Account Lockout

### Database Migration (`0001_008_login_attempts.sql`)
```sql
CREATE TABLE login_attempts (
    id SERIAL PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    email TEXT NOT NULL,
    ip_address TEXT NOT NULL,
    success BOOLEAN NOT NULL DEFAULT false,
    attempted_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE account_lockouts (
    id SERIAL PRIMARY KEY,
    user_id UUID UNIQUE NOT NULL REFERENCES users(id),
    locked_until TIMESTAMPTZ NOT NULL,
    failed_attempts INT NOT NULL DEFAULT 0,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Progressive Lockout Logic
- 1-5 attempts: No lockout, just track
- 6: Lock 15 min
- 7: Lock 30 min
- 8: Lock 1 hour
- 9: Lock 4 hours
- 10+: Lock 24 hours (max)

### New DB Functions
- `RecordLoginAttempt()`
- `IsAccountLocked()`
- `LockAccount()`
- `UnlockAccount()` (admin)
- `CleanupLoginAttempts()`

---

## 3. Audit Logging

### Database Migration (`0001_009_audit_log.sql`)
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
- New package: `backend/services/authservice/internal/audit/`
- Async logging option for performance
- 90-day retention with daily cleanup routine

---

## 4. JWT Private Key Encryption

### Approach: AES-256-GCM with env var master key

### Database Migration (`0001_010_key_encryption.sql`)
```sql
ALTER TABLE jwt_keys ADD COLUMN encryption_key_id TEXT;
ALTER TABLE jwt_keys ADD COLUMN encryption_iv TEXT;
ALTER TABLE jwt_keys ADD COLUMN encryption_tag TEXT;
```

### Implementation
- New package: `backend/swlib/encryption/`
- Master key: `ENCRYPTION_MASTER_KEY` (base64-encoded 256-bit)
- Generate key: `openssl rand -base64 32`
- Encrypt before storing, decrypt in-memory only for signing
- Supports gradual migration (existing keys remain readable)

---

## File Structure

```
backend/
├── swlib/
│   ├── ratelimit/
│   │   ├── ratelimit.go
│   │   ├── redis.go
│   │   └── ratelimit_test.go
│   └── encryption/
│       ├── encryption.go
│       ├── aes_gcm.go
│       └── encryption_test.go
└── services/authservice/
    ├── internal/
    │   ├── db/
    │   │   ├── login_attempts.go
    │   │   └── lockout.go
    │   ├── audit/
    │   │   ├── audit.go
    │   │   ├── postgres.go
    │   │   └── events.go
    │   └── server/
    │       └── authentication.go (modified)
    └── migrations/
        ├── 0001_008_login_attempts.sql
        ├── 0001_009_audit_log.sql
        └── 0001_010_key_encryption.sql
```

---

## Rollout Plan

### Week 1
- Day 1-2: Rate limiting (Redis integration)
- Day 3-4: Account lockout (migrations, logic)
- Day 5: Integration testing

### Week 2
- Day 1-2: Audit logging
- Day 3-4: JWT key encryption
- Day 5: E2E testing, docs

---

## Environment Variables

```bash
# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# Rate Limiting
RATE_LIMIT_IP_MAX=10
RATE_LIMIT_IP_WINDOW=60
RATE_LIMIT_EMAIL_MAX=5
RATE_LIMIT_EMAIL_WINDOW=300

# Account Lockout
LOCKOUT_MAX_ATTEMPTS=5
LOCKOUT_DURATION=900
LOCKOUT_PROGRESSIVE=true
LOCKOUT_MAX_DURATION=86400

# Audit
AUDIT_RETENTION_DAYS=90
AUDIT_ASYNC=true

# Encryption
ENCRYPTION_MASTER_KEY=<base64-256-bit>
```
