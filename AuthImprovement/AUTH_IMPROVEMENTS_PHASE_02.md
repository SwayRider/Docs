# Phase 2: High Impact Security Improvements

## Overview

Phase 2 addresses high-impact security gaps:
- **Cookie security hardening** - SameSite, Secure flag, optional encryption
- **TLS enforcement** - Database and gRPC connection encryption
- **Password breach detection** - HaveIBeenPwned integration
- **Password history** - Prevent password reuse

**Timeline**: 2 weeks
**Priority**: HIGH

---

## 1. Cookie Security Hardening

### Configuration
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| SameSite Mode | `COOKIE_SAMESITE` | strict |
| Secure Flag | `COOKIE_SECURE` | auto |
| Cookie Encryption | `COOKIE_ENCRYPTION_KEY` | (empty) |

### Changes
- Make SameSite configurable via env var (strict/lax/none)
- Auto-detect Secure flag from request context
- Optional AES-256 encryption for cookie values
- Local debugging: use `COOKIE_SAMESITE=lax` or `none`

### Implementation Details

**Modified**: `backend/swlib/http/cookies/cookie.go`

```go
// CookieOpts extended with sameSite
type CookieOpts struct {
    secure   bool
    domain   string
    ttl      time.Duration
    sameSite http.SameSite
}

// parseSameSite parses SameSite from environment
func parseSameSite(s string) http.SameSite {
    switch strings.ToLower(s) {
    case "strict":
        return http.SameSiteStrictMode
    case "lax":
        return http.SameSiteLaxMode
    case "none":
        return http.SameSiteNoneMode
    default:
        return http.SameSiteStrictMode // Default to strict
    }
}
```

---

## 2. TLS Enforcement

### Configuration
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Database SSL Mode | `DB_SSL_MODE` | disable |
| gRPC TLS Enabled | `GRPC_TLS_ENABLED` | false |
| gRPC TLS Cert | `GRPC_TLS_CERT_FILE` | (empty) |
| gRPC TLS Key | `GRPC_TLS_KEY_FILE` | (empty) |
| gRPC TLS CA | `GRPC_TLS_CA_FILE` | (empty) |

### Approach
Since you have Apache reverse proxy:
- **External TLS**: At Apache (already done)
- **Internal TLS**: Optional via env vars for database and gRPC

This gives flexibility without forcing complexity.

### Implementation Details

**1. Database TLS** (`backend/services/authservice/internal/db/postgres.go`)
```go
sslmode = d.cfg.SSLMode
if sslmode == "" {
    sslmode = os.Getenv("DB_SSL_MODE")
}
if sslmode == "" {
    if os.Getenv("ENV") == "production" {
        d.lg.Warnln("Database SSL mode not configured, defaulting to 'require'")
        sslmode = "require"
    } else {
        sslmode = "disable"
    }
}
```

**2. gRPC TLS** (`backend/swlib/app/grpc.go`)
```go
if GetConfigField[bool](a.cfg, "grpc-tls-enabled") {
    certFile := GetConfigField[string](a.cfg, "grpc-tls-cert-file")
    keyFile := GetConfigField[string](a.cfg, "grpc-tls-key-file")
    caFile := GetConfigField[string](a.cfg, "grpc-tls-ca-file")
    
    tlsConfig, err := loadTLSConfig(certFile, keyFile, caFile)
    // ...
    grpcOpts = append(grpcOpts, grpc.Creds(credentials.NewTLS(tlsConfig)))
}
```

---

## 3. Password Breach Detection (HaveIBeenPwned)

### How It Works
- Uses k-anonymity: only sends first 5 chars of SHA-1 hash
- Passwords never leave your server
- Privacy-preserving and free

### Configuration
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Enable Check | `HIBP_ENABLED` | true |
| API Timeout | `HIBP_TIMEOUT_MS` | 3000 |
| Minimum Count | `HIBP_MIN_COUNT` | 1 |

### New Package: `backend/swlib/hibp/`

```go
// Client checks passwords against HaveIBeenPwned
type Client struct {
    baseURL    string
    httpClient *http.Client
    enabled    bool
    minCount   int
}

// IsBreached checks if a password has been breached
// Returns (breached bool, count int, error)
func (c *Client) IsBreached(ctx context.Context, password string) (bool, int, error) {
    // Hash password with SHA-1
    hash := fmt.Sprintf("%x", sha1.Sum([]byte(password)))
    hash = strings.ToUpper(hash)
    
    // Split hash for k-anonymity
    prefix := hash[:5]
    suffix := hash[5:]
    
    // Query HIBP API with prefix only
    // Check if suffix appears in response
    // ...
}
```

### Integration Points
- **Registration**: Reject breached passwords
- **Password change**: Reject breached passwords
- **Password reset**: Reject breached passwords

### Error Handling
- **API timeout**: Log warning, allow password (fail open)
- **API error**: Log error, allow password (fail open)
- Ensures users aren't blocked if HIBP API is unavailable

---

## 4. Password History

### Database Migration (`0001_011_password_history.sql`)
```sql
CREATE TABLE password_history (
    id SERIAL PRIMARY KEY,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    password_hash TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_password_history_user_id ON password_history(user_id);
CREATE INDEX idx_password_history_created_at ON password_history(created_at);
```

### Configuration
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| History Size | `PASSWORD_HISTORY_SIZE` | 5 |

### New DB Functions (`backend/services/authservice/internal/db/password_history.go`)

```go
// AddToPasswordHistory stores a password hash in history
func (d *DB) AddToPasswordHistory(ctx context.Context, userID, passwordHash string) error

// GetPasswordHistory returns the most recent password hashes for a user
func (d *DB) GetPasswordHistory(ctx context.Context, userID string, limit int) ([]string, error)

// CleanupPasswordHistory removes old password history entries
func (d *DB) CleanupPasswordHistory(ctx context.Context, keepPerUser int) error

// CheckPasswordReuse checks if a password was recently used
func (d *DB) CheckPasswordReuse(ctx context.Context, userID, newPassword string) (bool, error)
```

### Behavior
- Stores last 5 password hashes per user
- Checks new password against history before accepting
- Daily cleanup removes excess entries
- Integrated into password change and reset flows

---

## File Structure

```
backend/
├── swlib/
│   ├── http/cookies/cookie.go (modified)
│   └── hibp/
│       ├── hibp.go
│       ├── client.go
│       └── hibp_test.go
├── services/authservice/
│   ├── internal/
│   │   ├── db/
│   │   │   ├── password_history.go (new)
│   │   │   └── postgres.go (modified)
│   │   └── server/
│   │       ├── registration.go (modified)
│   │       ├── change_password.go (modified)
│   │       └── password_reset.go (modified)
│   └── migrations/
│       └── 0001_011_password_history.sql (new)
└── swlib/app/grpc.go (modified)
```

---

## Environment Variables Summary

```bash
# Cookie Security
COOKIE_SAMESITE=strict
COOKIE_SECURE=auto
COOKIE_ENCRYPTION_KEY=

# TLS
DB_SSL_MODE=disable
GRPC_TLS_ENABLED=false
GRPC_TLS_CERT_FILE=
GRPC_TLS_KEY_FILE=
GRPC_TLS_CA_FILE=

# HIBP
HIBP_ENABLED=true
HIBP_TIMEOUT_MS=3000
HIBP_MIN_COUNT=1

# Password History
PASSWORD_HISTORY_SIZE=5
```

---

## Rollout Plan

### Week 3
- Day 1: Cookie security hardening
- Day 2: Optional cookie encryption
- Day 3-4: TLS enforcement
- Day 5: Testing

### Week 4
- Day 1-2: HIBP integration
- Day 3-4: Password history
- Day 5: Integration testing, documentation

---

## Testing Strategy

### Unit Tests
- Cookie SameSite configuration parsing
- HIBP client k-anonymity implementation
- Password history CRUD operations

### Integration Tests
- Cookie security headers in responses
- TLS connection establishment
- HIBP API integration (with mocked responses)
- Password reuse detection

### Manual Testing
- Verify cookies with browser dev tools
- Test TLS with `openssl s_client`
- Test breached password rejection
- Test password history enforcement
