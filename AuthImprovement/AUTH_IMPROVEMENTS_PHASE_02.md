# Phase 2: High Impact Security Improvements

## Overview

- **Cookie security hardening** — ✅ **DONE**: `Secure` flag auto-derived, `SameSite` configurable
- **TLS enforcement** — ⏳ **PENDING**
- **Password breach detection** — ⏳ **PENDING**
- **Password history** — ⏳ **PENDING**

---

## 1. Cookie Security Hardening

### `Secure` flag — ✅ DONE

Rather than a manual `COOKIE_SECURE` toggle, the `Secure` flag is now derived automatically from the `X-Forwarded-Proto` header (`authservice/internal/server/authentication.go`, `CookieHeaderMatcher`). No further work needed here.

### `SameSite` configurability — ✅ DONE

`COOKIE_SAMESITE` (env-only, default `strict`) now controls the refresh-token cookie's `SameSite` policy. Shipped with deliberate deviations from the original sketch:

- **Setter + parser in `swlib/http/cookies/cookie.go`, not env-reading in a constructor.** The package gained a package-level `defaultSameSite` (initialized to `Lax`), an exported `SetDefaultSameSite(http.SameSite)`, and an exported `ParseSameSite(string) (http.SameSite, error)`. `NewServerCookie`/`ClearCookie` now fall back to `defaultSameSite` instead of the hardcoded `SameSiteLaxMode` literal, so an explicit `CookieOpts.SetSameSite` still wins (which is why `swayrider-api`'s hardcoded `Strict` cookies are unaffected). This mirrors the existing `SetNamespace`/`COOKIE_NAMESPACE` pattern rather than reading process env inside the library.
- **Wired in `authservice/cmd/authservice/main.go`** next to the `COOKIE_NAMESPACE` block: `ParseSameSite(os.Getenv("COOKIE_SAMESITE"))`, warn on invalid, then `SetDefaultSameSite`. Unset → `strict`.
- **`none` deliberately rejected.** `SameSite=None` requires the `Secure` flag, which is derived per-request from `X-Forwarded-Proto` and can't be guaranteed, so a `none` cookie could be silently dropped by browsers. `ParseSameSite` accepts only `strict`/`lax` (case-insensitive) and returns an error for anything else; the caller logs and falls back to `strict`.
- **Default is `strict`, not `Lax`.** This is the intended hardening: the refresh cookie is only used in same-site fetch/XHR calls, so the stricter policy doesn't break login/refresh/logout (email verification and password reset use URL tokens, not this cookie).

Tests: `swlib/http/cookies/cookie_test.go` (`TestParseSameSite`, `TestSetDefaultSameSite`).

Optional AES-256 cookie-value encryption (`COOKIE_ENCRYPTION_KEY`) remains a stretch goal, not required for the `SameSite` fix.

---

## 2. TLS Enforcement — ⏳ PENDING

### Configuration (target)
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Database SSL Mode | `DB_SSL_MODE` | disable |
| gRPC TLS Enabled | `GRPC_TLS_ENABLED` | false |
| gRPC TLS Cert | `GRPC_TLS_CERT_FILE` | (empty) |
| gRPC TLS Key | `GRPC_TLS_KEY_FILE` | (empty) |
| gRPC TLS CA | `GRPC_TLS_CA_FILE` | (empty) |

`authservice/internal/db/postgres.go` still defaults `sslmode` to `disable` unconditionally — there's no `ENV=production` branch enforcing `require` today.

### Approach
- **External TLS**: terminated upstream (reverse proxy / `swayrider-api`)
- **Internal TLS**: optional via env vars for database and gRPC, so this doesn't force complexity on local dev

### Implementation Details

**1. Database TLS** (`authservice/internal/db/postgres.go`)
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

**2. gRPC TLS** (`swlib/app/grpc.go`)
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

## 3. Password Breach Detection (HaveIBeenPwned) — ⏳ PENDING

### How It Works
- k-anonymity: only the first 5 chars of the SHA-1 hash leave the server
- Passwords themselves never leave the server
- Privacy-preserving and free

### Configuration (target)
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Enable Check | `HIBP_ENABLED` | true |
| API Timeout | `HIBP_TIMEOUT_MS` | 3000 |
| Minimum Count | `HIBP_MIN_COUNT` | 1 |

### New Package: `swlib/hibp/`

```go
type Client struct {
    baseURL    string
    httpClient *http.Client
    enabled    bool
    minCount   int
}

// IsBreached checks if a password has been breached
// Returns (breached bool, count int, error)
func (c *Client) IsBreached(ctx context.Context, password string) (bool, int, error) {
    hash := fmt.Sprintf("%x", sha1.Sum([]byte(password)))
    hash = strings.ToUpper(hash)
    prefix := hash[:5]
    suffix := hash[5:]
    // Query HIBP API with prefix only, check if suffix appears in response
}
```

### Integration Points
- Registration, password change, password reset — all reject/warn on breached passwords

### Error Handling
- API timeout or error: log and fail open (allow the password) so an HIBP outage never blocks users

---

## 4. Password History — ⏳ PENDING

### Database Migration
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
Migration number must be chosen against the actual current sequence in `authservice/migrations/` at implementation time.

### Configuration (target)
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| History Size | `PASSWORD_HISTORY_SIZE` | 5 |

### New DB Functions (`authservice/internal/db/password_history.go`)

```go
func (d *DB) AddToPasswordHistory(ctx context.Context, userID, passwordHash string) error
func (d *DB) GetPasswordHistory(ctx context.Context, userID string, limit int) ([]string, error)
func (d *DB) CleanupPasswordHistory(ctx context.Context, keepPerUser int) error
func (d *DB) CheckPasswordReuse(ctx context.Context, userID, newPassword string) (bool, error)
```

### Behavior
- Stores last 5 password hashes per user, checked before accepting a new one
- Daily cleanup removes excess entries
- Integrated into password change and reset flows

---

## Environment Variables (pending items only)

```bash
# Cookie Security
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

## Testing Strategy (pending items only)

### Unit Tests
- HIBP client k-anonymity implementation
- Password history CRUD operations

### Integration Tests
- TLS connection establishment
- HIBP API integration (with mocked responses)
- Password reuse detection

### Manual Testing
- Test TLS with `openssl s_client`
- Test breached password rejection
- Test password history enforcement
