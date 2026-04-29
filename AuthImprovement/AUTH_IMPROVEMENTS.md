# Auth Improvements & Security Analysis

## Executive Summary

Analysis of the current authservice implementation with identified security gaps and improvement recommendations.

---

## Current Architecture

### Components
| Component | Port | Purpose |
|-----------|------|---------|
| gRPC Service | 8081 | Internal service-to-service communication |
| REST API | 8080 | HTTP API via grpc-gateway |
| Web Server | 8000 | Email verification/reset pages |

### Security Features
- RS256 JWT signing with automatic key rotation
- Argon2id password hashing (64MB memory, 1 iteration, 4 threads)
- Single-use refresh tokens with IP/user-agent binding
- Service client authentication with OAuth2 client credentials
- Endpoint-level security profiles (public, unverified, admin, service client)
- Password entropy validation (minimum 80 bits)

---

## Security Analysis

### 1. Password Security

**Strengths**
- Argon2id hashing: Winner of Password Hashing Competition, resistant to GPU and side-channel attacks
- Secure parameters: 64MB memory, 1 iteration, 4 threads - good balance
- Constant-time comparison: Uses `subtle.ConstantTimeCompare` to prevent timing attacks
- Password entropy validation: Minimum 80 bits entropy required

**Weaknesses**
- No password history: Users can reuse previous passwords
- No password expiration policy: Passwords never expire
- No breached password detection: No check against known compromised passwords

---

### 2. JWT Token Security

**Strengths**
- RS256 signing: Asymmetric algorithm, public keys for verification
- Automatic key rotation: New keys generated 3 days before expiration
- Advisory locks: Prevents race conditions during key rotation
- 2048-bit RSA: Minimum recommended key size
- 30-day key validity: Regular rotation limits exposure window

**Weaknesses**
- Private keys in database: RSA private keys stored in plaintext in PostgreSQL
- No key encryption at rest: Database compromise exposes signing keys
- Single key pair: Only one active signing key at a time (though old keys remain for verification)

---

### 3. Refresh Token Security

**Strengths**
- Single-use tokens: Old refresh token deleted before new one issued
- Token binding: Bound to IP address and user agent
- Hashed storage: Tokens stored as hashed values in database
- 64-byte random tokens: Cryptographically secure generation
- 30-day expiration: Reasonable lifetime for "remember me"

**Weaknesses**
- IP binding can break: Users behind NAT/load balancers share IPs
- User agent spoofing: User agent easily forged by attackers
- No token revocation list: Cannot revoke all tokens for a user (only one per user)
- Token stored in cookie without SameSite=Strict: Vulnerable to CSRF

---

### 4. Session Management

**Strengths**
- HttpOnly cookies: Prevents JavaScript access to refresh tokens
- SameSite=Lax: Partial CSRF protection
- Cookie namespacing: Prevents cookie collisions

**Weaknesses**
- Secure flag defaults to false: Cookies sent over HTTP unless explicitly configured
- No session timeout: Only refresh token expiration (30 days)
- No concurrent session control: Cannot limit sessions per user
- Cookie value base64 encoded: Not encrypted, visible in transit over HTTP

---

### 5. Authentication Flow

**Strengths**
- Separate public/verified/admin endpoints: Fine-grained access control
- Service client authentication: Separate OAuth2 client credentials flow
- Scope-based authorization: Service clients have scoped permissions

**Weaknesses**
- No rate limiting: Login attempts not throttled
- No account lockout: No lockout after failed attempts
- No MFA support: Single-factor authentication only
- No login anomaly detection: No detection of unusual login patterns

---

### 6. Email Verification

**Strengths**
- Token-based verification: 64-byte secure random tokens
- Time-limited tokens: Tokens expire
- Single-use tokens: One verification per token

**Weaknesses**
- No email rate limiting: Could spam verification emails
- Token in URL: Verification tokens visible in logs/browser history
- No email validation: Only format validation, not deliverability

---

### 7. Password Reset

**Strengths**
- Token-based reset: Separate from verification tokens
- Secure random tokens: 64-byte generation
- Time-limited: Tokens expire

**Weaknesses**
- No rate limiting: Could spam reset emails
- No user enumeration protection: Different responses for valid/invalid emails
- Token in URL: Reset tokens visible in logs

---

### 8. Service Client Security

**Strengths**
- Client credentials flow: Standard OAuth2 pattern
- Scope-based access: Granular permission control
- Hashed secrets: Client secrets stored as Argon2id hashes
- Admin-only creation: Only admins can create service clients

**Weaknesses**
- No secret rotation policy: Secrets never expire
- No audit logging: Service client usage not logged
- No IP restrictions: Service clients can authenticate from any IP

---

### 9. Infrastructure Security

**Strengths**
- PostgreSQL advisory locks: Prevents race conditions
- Database connection pooling: Standard sql.DB pooling
- Graceful shutdown: Proper cleanup on termination

**Weaknesses**
- No TLS enforcement: Database connections can use `sslmode=disable`
- No connection encryption: gRPC connections not encrypted by default
- No secrets management: Passwords in environment variables
- No health check authentication: Health endpoints publicly accessible

---

### 10. Logging & Monitoring

**Strengths**
- Structured logging: Uses custom logger with function/component context
- Debug logging for failures: Failed login attempts logged
- No sensitive data in logs: Passwords/hashes not logged

**Weaknesses**
- No audit trail: No persistent log of auth events
- No alerting: No alert on suspicious activity
- No metrics: No performance/security metrics

---

## Vulnerability Summary

| Category | Severity | Issue |
|----------|----------|-------|
| Authentication | HIGH | No rate limiting on login attempts |
| Authentication | HIGH | No account lockout after failed attempts |
| Session | MEDIUM | Refresh token cookie lacks SameSite=Strict |
| Session | MEDIUM | Secure flag defaults to false |
| JWT | HIGH | Private keys stored unencrypted in database |
| Password | MEDIUM | No breached password detection |
| Password | MEDIUM | No password history |
| Service Clients | MEDIUM | No secret rotation policy |
| Infrastructure | MEDIUM | Database can run without TLS |
| Monitoring | HIGH | No audit logging |
| Monitoring | MEDIUM | No anomaly detection |

---

## Recommended Improvements

### Priority 1: Critical Security

#### 1.1 Rate Limiting
- Add login attempt throttling (5 attempts per minute per IP)
- Add email-based rate limiting for registration/reset flows
- Implement exponential backoff for repeated failures

#### 1.2 Account Lockout
- Lock accounts after 5 failed login attempts
- Implement progressive lockout (15 min, 1 hour, 24 hours)
- Admin unlock capability
- Email notification on lockout

#### 1.3 Audit Logging
- Log all authentication events (login, logout, token refresh, password change)
- Log admin actions (user creation, account level changes)
- Log service client authentication
- Persist logs to database or external service
- Retention policy (90 days minimum)

#### 1.4 JWT Private Key Encryption
- Encrypt private keys at rest using database-level encryption or application-level encryption
- Use separate encryption key management (e.g., AWS KMS, HashiCorp Vault)
- Key rotation for encryption keys

---

### Priority 2: High Impact

#### 2.1 Cookie Security
- Set Secure flag based on HTTPS detection (already partially implemented)
- Change SameSite from Lax to Strict for refresh token cookies
- Consider encrypting cookie values (not just base64)

#### 2.2 TLS Enforcement
- Require TLS for database connections (`sslmode=require` minimum)
- Enforce TLS for gRPC communication
- Add TLS certificate validation

#### 2.3 Password Breach Detection
- Integrate with HaveIBeenPwned API (k-anonymity model)
- Check during registration and password change
- Warn users if password is compromised

#### 2.4 Password History
- Store last 5 password hashes
- Prevent password reuse
- Hash history with same Argon2id parameters

---

### Priority 3: Enhanced Security

#### 3.1 Multi-Factor Authentication
- TOTP support (Google Authenticator, Authy)
- Backup codes for account recovery
- Optional per-user enablement
- Admin enforcement option

#### 3.2 Session Management
- Add session timeout (configurable, default 15 minutes idle)
- Limit concurrent sessions per user
- Session listing and revocation for users
- Admin session termination

#### 3.3 User Enumeration Protection
- Consistent response times for valid/invalid emails
- Generic error messages ("If account exists, email sent")
- Rate limit password reset requests

#### 3.4 Secrets Management
- Move secrets from environment variables to secrets manager
- Rotate service client secrets automatically
- Audit secret access

---

### Priority 4: Operational

#### 4.1 Monitoring & Alerting
- Authentication metrics (success/failure rates, latency)
- Anomaly detection (unusual login patterns)
- Alerting on security events (mass failed logins, admin actions)
- Dashboard for auth health

#### 4.2 Key Rotation Improvements
- Support multiple active signing keys
- Graceful key rotation with overlap
- Automated key rotation notifications

#### 4.3 Service Client Improvements
- Secret expiration policy (90 days)
- IP allowlisting per service client
- Scope expiration (time-limited permissions)
- Usage quotas

---

## Implementation Roadmap

### Phase 1: Critical Security (Week 1-2)
- [ ] Rate limiting on login endpoint
- [ ] Account lockout mechanism
- [ ] Audit logging foundation
- [ ] JWT private key encryption

### Phase 2: High Impact (Week 3-4)
- [ ] Cookie security hardening
- [ ] TLS enforcement
- [ ] Password breach detection
- [ ] Password history

### Phase 3: Enhanced Security (Week 5-8)
- [ ] MFA support (TOTP)
- [ ] Session management improvements
- [ ] User enumeration protection
- [ ] Secrets management

### Phase 4: Operational (Week 9-10)
- [ ] Monitoring and alerting
- [ ] Key rotation improvements
- [ ] Service client hardening

---

## Keycloak Migration Alternative

If implementing all improvements is not feasible, migrating to Keycloak provides:
- Built-in rate limiting and brute force protection
- MFA support (TOTP, WebAuthn, SMS)
- Audit logging
- Admin UI
- Social login
- OIDC/OAuth2 compliance

See `KEYCLOAK_MIGRATION_ANALYSIS.md` in the `swayrider-keycloak` worktree for detailed comparison.

---

## Appendix: Relevant Code Locations

| Component | File |
|-----------|------|
| Password hashing | `backend/swlib/crypto/hashing.go` |
| JWT generation | `backend/swlib/jwt/jwt.go` |
| Refresh tokens | `backend/services/authservice/internal/model/refresh_token.go` |
| Cookie handling | `backend/swlib/http/cookies/cookie.go` |
| Auth interceptor | `backend/swlib/grpc/interceptors/authinterceptor.go` |
| Endpoint security | `backend/swlib/security/endpoint_profile.go` |
| Login flow | `backend/services/authservice/internal/server/authentication.go` |
| Key rotation | `backend/services/authservice/internal/db/jwt_keys.go` |
