# Phase 2: High Impact Security Improvements

## Overview

- **Cookie security hardening** — ✅ **DONE**: `Secure` flag auto-derived, `SameSite` configurable
- **TLS enforcement** — ⏳ **DEFERRED (to do later)**
- **Password breach detection** — ✅ **DONE** (implemented 2026-08-20; design record in §3)
- **Password history** — ✅ **DONE** (implemented 2026-08-20; design record in §4)

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

## 2. TLS Enforcement — ⏳ DEFERRED (to do later)

> **Deferred**: no work is planned for this item right now — it will be picked up at a later date. The design notes below are retained as-is for when it is taken on again.

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

## 3. Password Breach Detection (HaveIBeenPwned) — ✅ DONE

Implemented 2026-08-20. The plan below is retained as the design record; deviations from it are noted inline.

### How It Works
- k-anonymity: only the first 5 chars of the SHA-1 hash leave the server
- Passwords themselves never leave the server
- Privacy-preserving and free — the Pwned Passwords range API requires **no API key**

### Configuration (target)
| Parameter | Env Variable | Default |
|-----------|--------------|---------|
| Enable Check | `HIBP_ENABLED` | true |
| API Timeout | `HIBP_TIMEOUT_MS` | 3000 |
| Minimum Count | `HIBP_MIN_COUNT` | 1 |

### API Contract (verified against the v3 docs)
- `GET https://api.pwnedpasswords.com/range/{first-5-chars-of-uppercase-SHA-1}` — free, no API key
- Response is `text/plain`, one `{suffix}:{count}` pair per line; suffix matching is case-insensitive
- `User-Agent` header is **required** (missing → HTTP 403)
- Send `Add-Padding: true` so response size can't leak match status (defeats response-size side channels)
- Any non-200 response (403, 429 rate limit, 5xx) is treated as an error → caller fails open

### New Package: `swlib/hibp/`

```go
type Client struct {
    baseURL    string        // default https://api.pwnedpasswords.com
    httpClient *http.Client  // HIBP_TIMEOUT_MS applied here
    enabled    bool
    minCount   int
    l          *log.Logger
}

// New builds the client. enabled=false short-circuits every check to
// (false, 0, nil) so deployments can switch the feature off entirely.
func New(enabled bool, timeout time.Duration, minCount int, l *log.Logger) *Client

// IsBreached reports whether password has appeared in a breach at least
// minCount times. Only the first 5 chars of the uppercase SHA-1 leave the
// server. Any API error is returned to the caller, which fails open.
func (c *Client) IsBreached(ctx context.Context, password string) (bool, int, error)
```

Implementation notes:
- SHA-1, `strings.ToUpper`; `prefix := hash[:5]`, `suffix := hash[5:]`
- Request headers: `User-Agent: swayrider-authservice`, `Add-Padding: true`
- Scan the body line by line; match on uppercase suffix; `count >= minCount` → breached
- No new dependencies — stdlib `net/http` only. This is the first outbound-HTTP client in the repo, so it becomes the reference pattern
- Unit tests via `httptest.Server`: only the 5-char prefix appears in the request path (full hash never leaves the process), suffix match / no-match, count below `minCount`, non-200 → error, disabled client makes no network call, padding + UA headers present

### Server Integration (authservice)

Narrow interface in `internal/server` (mirrors the existing `MailSender`/`Database` pattern) and the same shape in `internal/web`:

```go
// internal/server — new file breached_checker.go
package server

type BreachedChecker interface {
    IsBreached(ctx context.Context, password string) (bool, int, error)
}
```

- `AuthServer` gains a `breached BreachedChecker` field; `NewAuthServer` takes it as a parameter
- `RegisterConfig` in `internal/web` gains the same field for the web registration form
- Shared construction in `cmd/authservice/main.go`: build one `*hibp.Client` from the three config fields and hand it to both `grpcAuthRegistrar` (→ `NewAuthServer`) and `startWebServer` (→ `RegisterConfig`)

### Integration Points and Ordering

A single helper keeps the three flows in sync:

```go
// Returns a gRPC InvalidArgument error when breached; on any HIBP API
// error it logs a warning and fails open (returns nil).
func (s *AuthServer) checkNotBreached(ctx context.Context, password string) error
```

| Flow | Where | Behavior on breach |
|------|-------|--------------------|
| `Register` (`registration.go`) | after entropy validation, before hashing | `codes.InvalidArgument` — non-enumerating, so a distinct error is safe |
| `ChangePassword` (`change_password.go`) | after old-password verification, before hashing | `codes.InvalidArgument` |
| `ResetPassword` (`password_reset.go`) | after token validation, before hashing | `codes.InvalidArgument` |
| Web `register` handler (`internal/web/register.go`) | after entropy validation, before hashing | re-render form with new i18n key `register_password_breached` + template string |
| Web `resetPassword` handler (`internal/web/reset_password.go`) | after entropy validation, before dbConn work | re-render form with new i18n key `reset_password_breached` + template string |

> **Deviation (implemented)**: the web `resetPassword` handler runs the breach check right after the entropy check, **before** token validation. The plan had it after token validation; the web handlers take a concrete `*db.DB` (no interface to stub), so the check had to precede the DB-backed token lookup to stay testable with a stub checker and a nil DB. It fails open either way, so an invalid token is still reported as such — the check just runs first.

Ordering rationale: cheap local checks (entropy, old password, reset token) run first so an HIBP network call is never wasted on a request that fails locally anyway; the HIBP call is the last pre-hash gate.

### Flutter Client Work (required — the rejection must surface as a distinct message)

Error text never reaches the app raw: the gateway (`swayrider-api/internal/handlers/auth.go`) sanitizes it and `AuthApiClient` matches on a stable `reason` field in the error body (`_errorReason` → typed exceptions). Today `register()` is the **only** method that parses `reason` (→ `WeakPasswordException`); `changePassword()` and `resetPassword()` only distinguish 401/200 and return a generic `HttpException` otherwise. So the client work spans six layers:

**1. Gateway `errBody`** (`swayrider-api/internal/handlers/auth.go`)
- Mirror the existing `weak_password` branch: when the gRPC message starts with the fixed prefix `password has appeared in a known data breach`, set `body["reason"] = "breached_password"` (HTTP 400 already comes from the `InvalidArgument` → `grpcStatus` mapping).
- Test in `swayrider-api/internal/handlers/errors_test.go`: prefix match → reason set; other errors → no reason.

**2. `AuthApiClient`** (`swayriderapp/lib/data/services/api/auth_api_client.dart`)
- New `BreachedPasswordException` (same shape as `WeakPasswordException`).
- `register()`: add `reason == 'breached_password'` branch next to `weak_password`.
- `changePassword()`: add a 400 + `_errorReason == 'breached_password'` branch (today it only handles 401).
- `resetPassword()`: same — add 400 + reason branch.
- **No repository change**: `AuthRepositoryRemote` returns `Error(error)` untouched for all three calls (verified), so the exception flows client → repository → viewmodel `command.result` → screen with no edits in between.

**3. Viewmodels** — one getter each, mirroring the existing `SignupViewModel.passwordTooWeak`:
```dart
bool get breachedPassword {
  final result = signup.result; // changePassword.result / resetPassword.result
  return result is Error && result.error is BreachedPasswordException;
}
```
- `SignupViewModel` (`signup_viewmodel.dart`) — getter added next to `passwordTooWeak`
- `ChangePasswordViewModel` (`change_password_viewmodel.dart`) — new; this viewmodel currently has no backend-error getter at all
- `NewPasswordViewModel` (`new_password_viewmodel.dart`) — new

**4. Screens** — one conditional per screen; all three already wrap the body in a `ListenableBuilder` on the command, so no extra rebuild wiring. Follow the **existing precedence pattern**: the generic-failure message must not render alongside the breach message. `SignupScreen` already does this for `invitationRequired`/`passwordTooWeak` by excluding them from `hasError`; extend it the same way:
```dart
// SignupScreen: add !widget.viewModel.breachedPassword to the hasError condition,
// then render the breach message in its own block, after the _passwordTooWeak block:
if (widget.viewModel.breachedPassword) ...[
  const SizedBox(height: Dimens.paddingVertical),
  ErrorMessage(text: localization.passwordBreached),
],
```
- `SignupScreen` (`signup_screen.dart`) — extend the existing `hasError` exclusion list and add the block
- `ChangePasswordScreen` (`change_password_screen.dart`) — compute `hasError` as `error && !breachedPassword` and add the block
- `NewPasswordScreen` (`new_password_screen.dart`) — same as change-password

**5. Localization** (`applocalization.dart` + `applocalization_en.dart` + `applocalization_nl.dart`)
- New getter in the base class: `String get passwordBreached => _get('passwordBreached');`
- en: `'passwordBreached': 'This password has appeared in a known data breach. Please choose a different one.'`
- nl: Dutch translation of the same string
- Web side (already covered in the integration table above): `register_password_breached` / `reset_password_breached` in `authservice/internal/web/i18n.go`

**6. Tests**
- `test/data/services/api/auth_api_client_test.dart` — for each of the three methods: 400 body with `reason: breached_password` → `BreachedPasswordException` (and the existing `weak_password` case still maps to `WeakPasswordException`)
- `test/ui/signup/view_models/signup_viewmodel_test.dart`, `change_password_viewmodel_test.dart`, `new_password_viewmodel_test.dart` — `Error(BreachedPasswordException())` → getter true; other errors → false
- No new widget tests: the repo has no screen-level widget tests (screens are covered via viewmodel tests), and each screen's change is a one-line conditional on the getter — keep it consistent

**Screens affected**: `SignupScreen`, `ChangePasswordScreen`, `NewPasswordScreen` (deep-link reset), web `/register`, web `/reset-password`. **Without this client work the password is still rejected** (the API refuses it) but the user sees only the generic failure message — same as any other `InvalidArgument`.

### Error Handling
- API timeout or error: log a warning and fail open (allow the password) so an HIBP outage never blocks users
- `HIBP_ENABLED=false`: skip the network call entirely, always allow

### Env Vars & Docs
- `authservice/env.example`: add the three `HIBP_*` vars with comments
- `authservice/README.md`: document the vars and the fail-open behavior

### Tests
- `swlib/hibp/hibp_test.go` — httptest-based (see implementation notes above)
- `internal/server/registration_test.go`, `change_password_test.go`, `password_reset_test.go` — stub `BreachedChecker`: breached → rejected; API error → allowed (fail open); disabled → allowed
- `internal/web` register + resetPassword handler tests — breached password re-renders the form with the error key
- `swayrider-api/internal/handlers/errors_test.go` — `errBody` sets `reason: breached_password` for the fixed message prefix, and no reason for other errors
- Flutter `auth_api_client_test.dart` — `reason == breached_password` maps to `BreachedPasswordException` for register, change-password, and reset-password; viewmodel tests for the three screens assert the `breachedPassword` getter (see the Flutter Client Work section above)

### Open Decisions
- **Resolved (2026-08-20):** the gateway matches on the fixed prefix `password has appeared in a known data breach` — `ErrBreachedPasswordPrefix` in `internal/server/breached_checker.go`, mirrored as `breachedPasswordPrefix` in `swayrider-api/internal/handlers/auth.go`.
- **Resolved (2026-08-20):** rejections now emit `auth.password_breached_rejected` audit events (`db.AuditPasswordBreachedRejected`) from `Register` (email only — no account exists yet), `ChangePassword`/`ResetPassword` (user ID + email), and the web register/reset forms (via `AuditWriter.Emit`, wired as `web.RegisterConfig.Audit`).

---

## 4. Password History — ✅ DONE

Implemented 2026-08-20. The plan below is retained as the design record; deviations from it are noted inline.

### How It Works
- A `password_history` table stores the **last N password hashes per user** (N = `PASSWORD_HISTORY_SIZE`, default 5), in the same Argon2id format as `users.password_hash`
- `ChangePassword` and `ResetPassword` **reject a new password that matches any of those hashes** (`InvalidArgument`) — a user cannot rotate back to a recent password
- The hash of **every newly-set password is recorded**: on registration (seeds history), on change, on reset, and for the bootstrapped admin — so the current password is always in history and a reset to the *current* password is also caught (ResetPassword today has no "must differ from old" check)
- History write failures are **log-only** (fail-open, like audit events); the reuse **check** also fails open on a DB error (log + allow)

### Database Migration — `authservice/migrations/0001_014_password_history.sql` (next free number)
```sql
-- +migrate Up
-- Stores the last N Argon2id password hashes per user so password change and
-- reset can reject reuse of recent passwords. See internal/db/password_history.go.
CREATE TABLE IF NOT EXISTS password_history (
    id            BIGSERIAL PRIMARY KEY,
    user_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    password_hash TEXT NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Covers both the per-user newest-N lookup/trim and the global cleanup sweep.
CREATE INDEX idx_password_history_user_id ON password_history (user_id, created_at DESC, id DESC);

-- +migrate Down
DROP TABLE password_history;
```

### Configuration — plumbed through `db.Config`, not `NewAuthServer`
| Parameter | Env Variable | Flag | Default |
|-----------|--------------|------|---------|
| History Size | `PASSWORD_HISTORY_SIZE` | `-password-history-size` | 5 |

- New consts in `cmd/authservice/main.go` (mirroring the HIBP block): `FldPasswordHistorySize`/`EnvPasswordHistorySize`/`DefPasswordHistorySize = 5` + `app.NewIntConfigField` entry
- `db.Config` (`internal/db/postgres.go`) gains `PasswordHistorySize int`; `dbCtor` in `main.go` reads it and stores it on `*db.DB`. **The size lives in the db layer**, so the server `Database` interface methods stay size-free and `NewAuthServer` gains no new parameter

### New DB File — `authservice/internal/db/password_history.go`

```go
// AddToPasswordHistory inserts a hash and trims the user's history to the
// configured size (newest N by created_at DESC, id DESC).
func (d *DB) AddToPasswordHistory(ctx context.Context, userID, passwordHash string) error

// CheckPasswordReuse reports whether newPassword matches any of the user's
// stored history hashes (fetches up to N, verifies with crypto.VerifyPassword).
func (d *DB) CheckPasswordReuse(ctx context.Context, userID, newPassword string) (bool, error)

// cleanupPasswordHistory removes every user's rows beyond the configured size
// (ROW_NUMBER() OVER (PARTITION BY user_id ...)); the hourly maintenance
// safety net for rows written before a size change.
func (d *DB) cleanupPasswordHistory(ctx context.Context) error
```

- Trim query: `DELETE FROM password_history WHERE user_id = $1 AND id NOT IN (SELECT id FROM password_history WHERE user_id = $1 ORDER BY created_at DESC, id DESC LIMIT $2)`
- `DoDatabaseMaintenance` (`internal/db/maintenance.go`) adds one `d.cleanupPasswordHistory(ctx)` call after the existing cleanups; no signature change (size comes from `d.passwordHistorySize`)

### Server Integration (authservice)

`Database` interface (`internal/server/database.go`) gains two methods; `mockDB` (`testhelpers_test.go`) gains `addToPasswordHistoryFn`/`checkPasswordReuseFn`:

```go
AddToPasswordHistory(ctx context.Context, userID, passwordHash string) error
CheckPasswordReuse(ctx context.Context, userID, newPassword string) (bool, error)
```

New helper in `internal/server` (mirrors `checkNotBreached`):

```go
// ErrPasswordReusedPrefix is the fixed message prefix the gateway matches on.
const ErrPasswordReusedPrefix = "password has been used before"

// checkNotReused rejects newPassword when it matches a recent history hash.
// On any DB error it logs and fails open (returns nil).
func (s *AuthServer) checkNotReused(ctx context.Context, userID, newPassword string) error
```

| Flow | Where | Behavior |
|------|-------|----------|
| `Register` (`registration.go`) | after `RegisterUser` succeeds (and the invite-only checks), before email send | seed: `AddToPasswordHistory(ctx, userid, hashedPassword)` — log-only on error |
| `ChangePassword` (`change_password.go`) | after old-password verification, **before** the HIBP check | `checkNotReused` → `InvalidArgument`; then after `UpdatePassword` succeeds, `AddToPasswordHistory` (log-only) |
| `ResetPassword` (`password_reset.go`) | after token validation + entropy, **before** the HIBP check | `checkNotReused` → `InvalidArgument`; after `UpdatePassword`, `AddToPasswordHistory` (log-only) |
| Web `register` (`internal/web/register.go`) | after `RegisterUser` succeeds | `dbConn.AddToPasswordHistory` (log-only) |
| Web `resetPassword` (`internal/web/reset_password.go`) | after token validation, before hashing; after `UpdatePassword` add to history | `dbConn.CheckPasswordReuse` → render form with i18n key `reset_password_reused`; log-only history add |
| `bootstrapAdmin` (`cmd/authservice/main.go`) | after `CreateAdminUser` | capture the returned userId and `dbConn.AddToPasswordHistory` the admin's initial hash (log-only) |

Ordering rationale: the reuse check is a cheap local Argon2id comparison and runs **before** the HIBP network call in both flows, so an HIBP request is never wasted on a password that fails locally anyway. The history write happens only after the new password is durably stored. A `ChangePassword` to the current password is already rejected by the existing "must differ" check, and `ResetPassword`'s reuse of the current password is caught because the current hash is always in history.

### Gateway (`swayrider-api/internal/handlers/auth.go`)

- New const `passwordReusedPrefix = "password has been used before"` (mirror of `breachedPasswordPrefix`; must match `ErrPasswordReusedPrefix` exactly)
- `errBody`: add `case strings.HasPrefix(s.Message(), passwordReusedPrefix): body["reason"] = "password_reused"` (HTTP 400 already comes from `InvalidArgument`)
- Test in `errors_test.go`: prefix → `password_reused`; other errors → no reason

### Flutter Client Work

Same 3-hop plumbing as breach detection — reuse can only happen on change/reset, **not** register, so this is narrower:

1. **`AuthApiClient`** — new `PasswordReusedException`; add a 400 + `reason == 'password_reused'` branch to `changePassword()` and `resetPassword()` only
2. **Viewmodels** — `passwordReused` getter (same shape as `passwordBreached`) on `ChangePasswordViewModel` and `NewPasswordViewModel`; no change to `SignupViewModel`
3. **Screens** — `ChangePasswordScreen` and `NewPasswordScreen`: extend the `hasError` exclusion and add a block rendering `localization.passwordReused`; `SignupScreen` untouched
4. **Localization** — `passwordReused` getter in `applocalization.dart` + en/nl strings (`'This password has been used recently. Please choose a different one.'`)
5. **Web i18n** — `reset_password_reused` key in `authservice/internal/web/i18n.go` (en + nl)

### Tests
- `internal/server/breached_checker_test.go`-style additions (or a new `password_history_test.go` in `internal/server`): mockDB with `checkPasswordReuseFn` → reused → `InvalidArgument` + `UpdatePassword`/`AddToPasswordHistory` not called; DB error → fail open; `Register` seeds history after `RegisterUser`; `ChangePassword`/`ResetPassword` add to history after a successful update
- `swayrider-api/internal/handlers/errors_test.go` — `reason: password_reused` for the fixed prefix
- Flutter `auth_api_client_test.dart` — 400 + `password_reused` → `PasswordReusedException` for change/reset; viewmodel tests for the two getters
- **No web-handler test for reuse**: the web `resetPassword` reuse check needs `GetUserByID` + the concrete `*db.DB` (same constraint as the breach check — see the deviation note in §3); covered indirectly via the gRPC-path server tests

### Env Vars & Docs
- `authservice/env.example` + `authservice/README.md`: document `PASSWORD_HISTORY_SIZE` and the reuse-rejection behavior

### Open Decisions
- **Resolved (2026-08-20):** the gateway matches on the fixed prefix `password has been used before` — `ErrPasswordReusedPrefix` in `internal/server/password_history.go`, mirrored as `passwordReusedPrefix` in `swayrider-api/internal/handlers/auth.go`.
- **Resolved (2026-08-20):** reuse rejections now emit `auth.password_reuse_rejected` audit events (`db.AuditPasswordReuseRejected`) from `ChangePassword`/`ResetPassword` and the web reset form — consistent with the breach-rejection audit.

> **Implementation notes (2026-08-20)**: no deviations from the plan. The reuse check runs before the HIBP check in both change and reset (cheap local first); the web `resetPassword` reuse check sits after token validation since it needs the user row from the concrete `*db.DB` (and is therefore covered by the gRPC-path tests, not a web handler test — as planned). Audit events on rejection were added the same day (see Open Decisions).

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

```

---

## Testing Strategy (pending items only)

### Integration Tests
- TLS connection establishment

### Manual Testing
- Test TLS with `openssl s_client`
