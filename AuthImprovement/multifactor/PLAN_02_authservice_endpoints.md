# PLAN 02 — protos + authservice endpoints + login flow

**Layer:** `protos/auth/v1`, `authservice/internal/server` (+ `cmd/authservice/main.go` wiring, QR rendering).
**Depends on:** Plan 1 (schema + db layer + `swlib/totp`).

---

## 1. Proto changes — `protos/auth/v1/auth.proto`

`LoginResponse` gains two fields (backward-compatible addition):

```proto
message LoginResponse {
  string access_token = 1;
  string refresh_token = 2;
  bool   mfa_required = 3;  // true when the account has MFA enabled
  string mfa_token    = 4;  // pending-login challenge token, only when mfa_required
}
```

New RPCs (all with `google.api.http` annotations, consistent with existing style):

| RPC | HTTP | Auth | Purpose |
|-----|------|------|---------|
| `SetupMFA` | `POST /api/v1/auth/mfa/setup` | Unverified | Start enrollment → secret + otpauth URL + QR PNG |
| `EnableMFA` | `POST /api/v1/auth/mfa/enable` | Unverified | Verify one code → enable → issue backup codes |
| `DisableMFA` | `POST /api/v1/auth/mfa/disable` | Unverified | Disable (requires password) |
| `GetMFAStatus` | `GET /api/v1/auth/mfa/status` | Unverified | Enabled? |
| `VerifyMFA` | `POST /api/v1/auth/mfa/verify` | Public | Exchange challenge + code for tokens |
| `GenerateBackupCodes` | `POST /api/v1/auth/mfa/backup-codes` | Unverified | Regenerate (invalidates old) |

```proto
message SetupMFARequest {}
message SetupMFAResponse {
  string secret          = 1;  // base32, shown once + grouped in the app
  string otpauth_url     = 2;
  string qr_png_base64   = 3;  // server-rendered QR of otpauth_url
}
message EnableMFARequest { string code = 1; }
message EnableMFAResponse { repeated string backup_codes = 1; }
message DisableMFARequest { string password = 1; }
message DisableMFAResponse {}
message GetMFAStatusRequest {}
message GetMFAStatusResponse { bool enabled = 1; }
message VerifyMFARequest { string mfa_token = 1; string code = 2; }
message VerifyMFAResponse { string access_token = 1; string refresh_token = 2; }
message GenerateBackupCodesRequest { string password = 1; }
message GenerateBackupCodesResponse { repeated string backup_codes = 1; }
```

Then `cd protos && make` (regenerates `auth.pb.go`, `auth_grpc.pb.go`, `auth.pb.gw.go`).

**VerifyMFA and cookies:** grpc-gateway's `CookieForwarder` (Plan 2 §4) must handle `VerifyMFAResponse` the same as `LoginResponse`. Note the service's own REST surface is internal-only; the external path is the gateway (Plan 3) which manages its own cookies.

---

## 2. Security registration — `internal/server/server.go` `init()`

```go
security.UnverifiedEndpoint("/auth.v1.AuthService/SetupMFA")
security.UnverifiedEndpoint("/auth.v1.AuthService/EnableMFA")
security.UnverifiedEndpoint("/auth.v1.AuthService/DisableMFA")
security.UnverifiedEndpoint("/auth.v1.AuthService/GetMFAStatus")
security.UnverifiedEndpoint("/auth.v1.AuthService/GenerateBackupCodes")
security.PublicEndpoint("/auth.v1.AuthService/VerifyMFA")
```

---

## 3. Server config — `MFAConfig`

New struct + `NewAuthServer` parameter (mirrors how `breached BreachedChecker` was added):

```go
// internal/server/mfa_config.go
type MFAConfig struct {
    Enabled               bool          // MFA_ENABLED
    CodeLength            int           // MFA_CODE_LENGTH
    TimeStep              time.Duration // MFA_TIME_STEP
    GracePeriod           int           // MFA_GRACE_PERIOD
    BackupCodeCount       int           // MFA_BACKUP_CODES
    ChallengeTTL          time.Duration // MFA_CHALLENGE_TTL_SECS
    ChallengeMaxAttempts  int           // MFA_CHALLENGE_MAX_ATTEMPTS
    LockoutMaxAttempts    int           // MFA_LOCKOUT_THRESHOLD
    LockoutWindow         time.Duration // MFA_LOCKOUT_WINDOW_SECS
    LockoutDuration       time.Duration // MFA_LOCKOUT_DURATION_SECS
}

// totpConfig() derives swlib/totp.Config from the MFAConfig.
```

`cmd/authservice/main.go` `grpcAuthRegistrar` builds it from the Plan 1 config fields and passes it to `NewAuthServer`. `AuthServer` gains an `mfa MFAConfig` field.

---

## 4. Endpoints — new file `internal/server/mfa.go`

Fixed error prefixes (contract for the gateway, mirroring `ErrBreachedPasswordPrefix`):

```go
const (
    ErrMFADisabledPrefix   = "mfa is disabled"           // global switch off
    ErrMFAAlreadySetupPrefix = "mfa is already enabled"  // EnableMFA on an enabled account
    ErrMFANotSetupPrefix   = "mfa is not set up"         // Enable/Disable without a secret row
    ErrInvalidMFACodePrefix = "invalid authentication code" // TOTP/backup-code rejection
)
```

All management endpoints start with a `checkMFAEnabled()` guard: `MFAConfig.Enabled == false` → `FailedPrecondition` with `ErrMFADisabledPrefix` (fails **closed** for management; login simply bypasses the MFA step when the switch is off — see §5).

- **`SetupMFA`** (`getUserFromClaims`): if `GetMFAStatus` is already true → `FailedPrecondition` (`ErrMFAAlreadySetupPrefix`). Generate secret (`totp.GenerateSecret(20)`), build `totp.GenerateOTPAuthURL(secret, email, "SwayRider")`, render QR PNG from the URL, encrypt secret via the db `CreateMFASecret` (upsert, `enabled=false`). Return secret + URL + `base64.StdEncoding` of the PNG. Audit `auth.mfa_setup_started`.
  - **QR rendering:** small helper `renderQRPNG(otpauthURL string) ([]byte, error)` in `internal/server` (or `internal/web`-adjacent util) using the Plan-verified QR lib (`github.com/skip2/go-qrcode` incumbent; `github.com/yeqown/go-qrcode/v2` if skip2 proves stale). ~256px, medium error correction. Verify lib choice and add to `authservice/go.mod` **at implementation time**.
- **`EnableMFA`**: guard (already-enabled → `FailedPrecondition`); `GetMFASecret` (absent → `FailedPrecondition` `ErrMFANotSetupPrefix`); `totp.Validate(secret, req.Code, now, cfg)`; invalid → `InvalidArgument` `ErrInvalidMFACodePrefix`. Valid → `EnableMFA`; generate `BackupCodeCount` codes (`totp.GenerateBackupCodes`), Argon2id-hash each (`crypto.CalculatePasswordHash`), `StoreBackupCodeHashes`, return plaintext codes. Audit `auth.mfa_enabled` + `auth.mfa_backup_codes_generated`. **The plaintext secret is never returned again after this point.**
- **`DisableMFA`**: `getUserFromClaims`; verify `req.Password` against the user's hash (`crypto.VerifyPassword`) — wrong → `Unauthenticated` (uniform "invalid password" — no enumeration signal). Correct → `DisableMFA` (deletes row + backup codes). Audit `auth.mfa_disabled`.
- **`GetMFAStatus`**: `GetMFAStatus` bool.
- **`VerifyMFA`**: (public) `sha256(req.MfaToken)` → `GetMFAChallenge`; absent/expired → `Unauthenticated` "invalid or expired mfa token". `IsAttemptLocked(ScopeMFA, userID)` → `Unauthenticated` (uniform message). Load secret (`GetMFASecret`) **and** verify the code:
  - TOTP path: `totp.Validate(secret, req.Code, now, cfg)`.
  - Backup-code path: lowercase-normalize input, hash, `ConsumeBackupCode` (atomic; `false` = already used/absent).
  - Failure: `IncrementMFAChallengeAttempts`; when attempts ≥ `ChallengeMaxAttempts` → `ConsumeMFAChallenge` (kill the challenge); `RecordAttemptResult(ScopeMFA, userID, false, LockoutMaxAttempts, LockoutWindow, LockoutDuration)`; return `Unauthenticated` with `ErrInvalidMFACodePrefix`. Audit `auth.mfa_verify_failed`.
  - Success: `ConsumeMFAChallenge`, `RecordAttemptResult(ScopeMFA, userID, true, ...)`, then `createAuthTokens(ctx, user, origIp, userAgent)` exactly like `Login` (single-use refresh token etc.). Audit `auth.mfa_verified`.
- **`GenerateBackupCodes`**: `getUserFromClaims`; must be enabled (else `FailedPrecondition`); verify `req.Password` (uniform `Unauthenticated` on mismatch); generate + hash new codes, `StoreBackupCodeHashes` (replaces old), return plaintext once. Audit `auth.mfa_backup_codes_generated`.

---

## 5. Login flow change — `internal/server/authentication.go`

Insert **after** `recordLoginAttempt(ctx, identifier, true)` / `auditLoginSuccess` and **before** `createAuthTokens`:

```go
// MFA gate: when the account has MFA enabled, do not issue tokens yet.
// Issue a short-lived single-use challenge token instead; the caller
// exchanges it (plus a TOTP/backup code) via VerifyMFA.
if s.mfa.Enabled {
    enabled, err := s.DB().GetMFAStatus(ctx, u.ID)
    if err != nil {
        return nil, status.Error(codes.Internal, "internal error")
    }
    if enabled {
        rawToken, err := s.createMFAChallenge(ctx, u.ID) // 32 random bytes; SHA-256 stored; TTL = ChallengeTTL
        if err != nil {
            return nil, status.Error(codes.Internal, "internal error")
        }
        return &authv1.LoginResponse{MfaRequired: true, MfaToken: rawToken}, nil
    }
}
```

Notes:

- The MFA challenge lookup is **not** an account-enumeration signal: it runs only after a *correct* password, and the response shape is uniform for MFA/non-MFA accounts (the app branches on `mfa_required`).
- `Refresh` is **unchanged** — the refresh token was issued only after a completed MFA login.
- `remember-me` gRPC header: not set on the pending response (no tokens yet); `VerifyMFA` needs to forward `remember_me`. Add `remember_me` to `VerifyMFARequest` and set the header in `VerifyMFAResponse` path (or have the gateway pass it through — decide in Plan 3; simplest is `VerifyMFARequest.remember_me = 3`).

## 6. CookieForwarder — `internal/server/authentication.go`

`VerifyMFAResponse` joins the cookie-set switch; guard the set-cookie branch with `if token != ""` so a hypothetical empty-token response never sets an empty cookie. (Plan 3's gateway does its own cookie handling; this keeps the service's internal REST surface consistent.)

## 7. Audit events — `internal/server/audit.go` + `internal/db/audit.go`

Server helpers + db event constructors, following `auditBreachedPasswordRejected` / `db.AuditPasswordBreachedRejected`:

- `auth.mfa_setup_started` (userID, email)
- `auth.mfa_enabled` (userID, email)
- `auth.mfa_disabled` (userID, email)
- `auth.mfa_verified` (userID)
- `auth.mfa_verify_failed` (userID, reason: `invalid_code` | `challenge_expired` | `locked`)
- `auth.mfa_backup_codes_generated` (userID)

## 8. Database interface + mock — `internal/server/database.go`, `testhelpers_test.go`

`Database` gains: `GetMFASecret`, `GetMFAStatus`, `CreateMFASecret`, `EnableMFA`, `DisableMFA`, `StoreBackupCodeHashes`, `ConsumeBackupCode`, `CreateMFAChallenge`, `GetMFAChallenge`, `IncrementMFAChallengeAttempts`, `ConsumeMFAChallenge`. `mockDB` gains the matching `*Fn` fields + zero-value defaults (mirroring `addToPasswordHistoryFn`/`checkPasswordReuseFn`).

## 9. Tests

- `internal/server/mfa_test.go` (mockDB):
  - Setup: returns 32-char secret + otpauth URL + decodable PNG; stores encrypted; already-enabled → `FailedPrecondition`; global switch off → `FailedPrecondition`.
  - Enable: valid code → enabled + backup codes returned (count = `BackupCodeCount`, Crockford alphabet); invalid code → `InvalidArgument` with prefix; no secret row → `FailedPrecondition`.
  - Disable: wrong password → `Unauthenticated`; correct → row + codes deleted.
  - Verify: valid TOTP → token pair (refresh stored via mockDB); valid backup code → tokens, second use of same code fails (single-use); unknown/expired challenge → `Unauthenticated`; attempts reach max → challenge consumed + `RecordAttemptResult` failure recorded; locked scope → `Unauthenticated`; success clears throttle.
  - Backup codes regen: old hashes replaced, new codes returned.
- `internal/server/authentication_test.go` additions: MFA-enabled account login → `MfaRequired` + token, **no** `createAuthTokens` call (assert mockDB refresh-token fn not invoked); MFA-disabled account → unchanged token pair; global switch off with MFA enabled on account → tokens (bypass).
- `internal/server/audit_test.go` additions: the six new event types emitted on their paths.
- CookieForwarder: `VerifyMFAResponse` sets a cookie; `LoginResponse` with empty refresh token sets none.

## Definition of done (Plan 2)

- Protos regenerated; `go build ./...` green in `protos` and `authservice`.
- All six endpoints + login branch implemented; server unit tests green; audit + throttle wiring in place.
- `authservice/README.md` documents the new endpoints, env vars, and the MFA login flow.
- No gateway, authclient, or app changes yet (they land in Plan 3+).
