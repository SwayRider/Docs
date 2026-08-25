# PLAN 01 — TOTP core + config + schema + db layer

**Layer:** `swlib/totp` (new package), `authservice` config, migration, `internal/db`.
**Depends on:** nothing. **Deliverable:** the crypto core, schema, and persistence for MFA, with no endpoint exposed yet.

---

## 1. `swlib/totp` (new package, stdlib-only)

New directory `swlib/totp/` with `totp.go` (+ `totp_test.go`). Pure stdlib (`crypto/hmac`, `crypto/sha1`, `crypto/rand`, `encoding/base32`, `crypto/subtle`) — mirrors the dependency-free precedent of `swlib/hibp`. **Do not add the QR dependency here** (see §3); the shared `swlib` module stays dependency-light.

```go
package totp

// Config tunes TOTP generation/validation. Zero values are replaced with
// the defaults below.
type Config struct {
    SecretSize  int           // random secret bytes; default 20
    CodeLength  int           // digits; default 6
    TimeStep    time.Duration // default 30s
    GracePeriod int           // accept ±N windows around the current one; default 1
}

// GenerateSecret returns SecretSize random bytes as unpadded, uppercase
// base32 (RFC 4648). crypto/rand only.
func GenerateSecret(size int) (string, error)

// GenerateCode computes the RFC 6238 code for secret at time t.
func GenerateCode(secret string, t time.Time, cfg Config) (string, error)

// Validate reports whether code matches secret within the current window
// plus GracePeriod windows before/after. Comparison is constant-time
// (crypto/subtle) against each candidate window.
func Validate(secret, code string, t time.Time, cfg Config) (bool, error)

// GenerateOTPAuthURL builds otpauth://totp/{issuer}:{account}?secret=...&issuer=...
// Account and issuer are percent-encoded. (This is the sketch's
// GenerateQRCodeURL, renamed: it returns the URL string, not a QR.)
func GenerateOTPAuthURL(secret, account, issuer string) string

// GenerateBackupCodes returns n random codes of the given length using the
// Crockford base32 alphabet (0-9 A-Z minus I, L, O, U — no ambiguous chars),
// each grouped for readability and returned without spaces.
func GenerateBackupCodes(n, length int) ([]string, error)
```

Implementation notes:

- RFC 6238: `HMAC-SHA1(key = base32decode(secret), counter = floor(t / TimeStep))`; dynamic truncation (`offset = mac[19] & 0x0f`, `bin = (mac[offset]&0x7f)<<24 | ... `); code = `bin % 10^CodeLength`, zero-padded to `CodeLength`.
- Secret bytes → base32 **without padding**, uppercase. 20 bytes → 32 chars.
- `GenerateCode` takes `time.Time` (not `time.Now()`) so tests can pin the clock; `Validate` also takes `t`.
- Backups codes must be unambiguous to type: Crockford base32 alphabet `0123456789ABCDEFGHJKMNPQRSTVWXYZ`.
- Guard: `CodeLength` clamped to 1–8 (TOTP truncation supports up to 8 digits; the phase doc's 6 is the default).

Tests (`totp_test.go`):

- **RFC 6238 Appendix B vectors** (HMAC-SHA1, 8-digit): seed `12345678901234567890`, the `T=59, 1111111109, 1111111111, 1234567890, 2000000000, 20000000000` cases.
- 6-digit default: derive expected values from the same vectors.
- `Validate` accepts current window and ±GracePeriod; rejects older/newer; rejects a wrong code in an otherwise valid window.
- Constant-time path exercised (structural — assert `crypto/subtle.ConstantTimeCompare` usage is not required, but ensure no early string compare in `Validate`).
- `GenerateSecret`: correct length, valid base32, uniqueness across calls.
- `GenerateBackupCodes`: count, length, alphabet membership (no I/L/O/U), uniqueness.
- `GenerateOTPAuthURL`: correct scheme/host/params, percent-encoding of `:` and `@` in account.

---

## 2. Config (`authservice/cmd/authservice/main.go`)

Add consts + `app.New*ConfigField` entries mirroring the HIBP block:

```go
FldMfaEnabled               = "mfa-enabled"                 // EnvMfaEnabled = "MFA_ENABLED",                 Def = true
FldMfaCodeLength            = "mfa-code-length"             // "MFA_CODE_LENGTH",                             6
FldMfaTimeStepSecs          = "mfa-time-step-secs"          // "MFA_TIME_STEP",                               30
FldMfaGracePeriod           = "mfa-grace-period"            // "MFA_GRACE_PERIOD",                            1
FldMfaBackupCodes           = "mfa-backup-codes"            // "MFA_BACKUP_CODES",                            10
FldMfaChallengeTtlSecs      = "mfa-challenge-ttl-secs"      // "MFA_CHALLENGE_TTL_SECS",                      300
FldMfaChallengeMaxAttempts  = "mfa-challenge-max-attempts"  // "MFA_CHALLENGE_MAX_ATTEMPTS",                  5
FldMfaLockoutThreshold      = "mfa-lockout-threshold"       // "MFA_LOCKOUT_THRESHOLD",                       5
FldMfaLockoutWindowSecs     = "mfa-lockout-window-secs"     // "MFA_LOCKOUT_WINDOW_SECS",                     900
FldMfaLockoutDurationSecs   = "mfa-lockout-duration-secs"   // "MFA_LOCKOUT_DURATION_SECS",                   900
```

Plumbed to the server in **Plan 2** (a new `server.MFAConfig` struct passed to `NewAuthServer`); this plan only registers the fields.

---

## 3. Migration `authservice/migrations/0001_015_mfa.sql` (next free number)

```sql
-- +migrate Up
-- TOTP second-factor enrollment. secret holds an AES-256-GCM blob (base64)
-- keyed by ENCRYPTION_MASTER_KEY; secret_key_id is the KeyRing fingerprint
-- so ENCRYPTION_MASTER_KEY_PREVIOUS rotation keeps working (mirrors jwt_keys).
CREATE TABLE IF NOT EXISTS user_mfa (
    id            BIGSERIAL PRIMARY KEY,
    user_id       UUID UNIQUE NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    enabled       BOOLEAN NOT NULL DEFAULT false,
    secret        TEXT NOT NULL,
    secret_key_id TEXT,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Single-use backup codes, Argon2id-hashed. Codes are consumed atomically.
CREATE TABLE IF NOT EXISTS mfa_backup_codes (
    id         BIGSERIAL PRIMARY KEY,
    user_id    UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash  TEXT NOT NULL,
    used       BOOLEAN NOT NULL DEFAULT false,
    used_at    TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_mfa_backup_codes_user ON mfa_backup_codes (user_id);

-- Pending-login challenge tokens: raw token never stored, only SHA-256.
-- One live challenge per user (new login deletes the previous one).
CREATE TABLE IF NOT EXISTS mfa_challenges (
    id          BIGSERIAL PRIMARY KEY,
    user_id     UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash  TEXT NOT NULL,
    attempts    INT NOT NULL DEFAULT 0,
    valid_until TIMESTAMPTZ NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_mfa_challenges_token ON mfa_challenges (token_hash);

-- +migrate Down
DROP TABLE mfa_challenges;
DROP TABLE mfa_backup_codes;
DROP TABLE user_mfa;
```

---

## 4. db layer — `authservice/internal/db/mfa.go`

Encryption mirrors `jwt_keys.go` (`encodeForStorage`/`decodeFromStorage` use `ring.EncryptCurrent` + `ring.Decrypt(blob, keyID)`); add the same pattern here with local helpers (e.g. `encryptSecret`/`decryptSecret`) so `user_mfa.secret` round-trips through the existing `d.keyRing`. `nil` ring → store plaintext (defensive/test seam, same as jwt_keys).

```go
// MFAUser is the decrypted row state.
type MFAUser struct {
    UserID  string
    Enabled bool
    Secret  string // plaintext, only ever returned during SetupMFA
}

// Upsert: one user_mfa row per user. Store the encrypted secret; on
// re-setup the previous secret and backup codes are replaced.
func (d *DB) CreateMFASecret(ctx context.Context, userID, encryptedSecret, keyID string) error

func (d *DB) GetMFASecret(ctx context.Context, userID string) (*MFAUser, error) // ErrNoMFARecord when absent
func (d *DB) GetMFAStatus(ctx context.Context, userID string) (bool, error)     // false when no row
func (d *DB) EnableMFA(ctx context.Context, userID string) error                // set enabled = true
func (d *DB) DisableMFA(ctx context.Context, userID string) error               // delete row + backup codes

// Replaces all backup codes for the user (delete + insert new hashes).
func (d *DB) StoreBackupCodeHashes(ctx context.Context, userID string, hashes []string) error

// Atomic single-use consume: UPDATE ... WHERE user_id = $1 AND code_hash = $2
// AND used = false SET used = true, used_at = now() RETURNING id; reports
// whether a row was claimed.
func (d *DB) ConsumeBackupCode(ctx context.Context, userID, codeHash string) (bool, error)

// Deletes any prior challenge for the user, then inserts the new one.
func (d *DB) CreateMFAChallenge(ctx context.Context, userID, tokenHash string, validUntil time.Time) error

func (d *DB) GetMFAChallenge(ctx context.Context, tokenHash string) (*model.MFAChallenge, error)
func (d *DB) IncrementMFAChallengeAttempts(ctx context.Context, tokenHash string) (int, error)
func (d *DB) ConsumeMFAChallenge(ctx context.Context, tokenHash string) error
```

- New errors in `internal/db/errors.go`: `ErrNoMFARecord`, `ErrNoMFAChallengeFound` (mirror `ErrNoPasswordResetTokenFound`).
- New model `model.MFAChallenge { UserID, TokenHash, Attempts int, ValidUntil time.Time }` in `internal/model`.
- **Maintenance** (`internal/db/maintenance.go`): add `d.cleanupMFAChallenges(ctx)` (delete `valid_until < now()`) to `DoDatabaseMaintenance` — no signature change.
- **Throttle scope** (`internal/db/throttle.go`): add `ScopeMFA ThrottleScope = "mfa"` to the const block. (Consumed in Plan 2.)
- **Testable helper**: keep the encrypt/decrypt secret helpers standalone like `jwt_keys_encoding_test.go` does; add `mfa_encoding_test.go` (round-trip with a real `KeyRing`, nil-ring plaintext path, wrong-key failure).

## Definition of done (Plan 1)

- `swlib/totp` implemented; `go test ./...` green in `swlib` (RFC 6238 vectors included).
- Migration `0001_015` applies cleanly (`cd authservice && make migrate-status` / `migrate-up` on a scratch DB).
- `internal/db/mfa.go` compiles with the new errors/model; maintenance + throttle scope updated; `mfa_encoding_test.go` green.
- Config fields registered in `main.go`; `env.example` gains the `MFA_*` vars.
- No server endpoints, no gateway, no app changes yet.
