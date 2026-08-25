# PLAN 03 — authclient wrapper + API gateway

**Layer:** `grpcclients/authclient`, `swayrider-api`.
**Depends on:** Plan 2 (new RPCs + `LoginResponse` fields exist in `protos`).

---

## 1. `grpcclients/authclient/client.go`

**Breaking change — `Login` signature** (grep for `\.Login(` across `swayrider-api`, `swctl`, and any other consumer at implementation time; all callers move in this plan):

```go
func (c Client) Login(
    email, password string,
    rememberMe bool,
    info ...ClientInfo,
) (
    accessToken string,
    refreshToken string,
    mfaRequired bool,
    mfaToken string,
    err error,
)
```

New methods (each follows the existing `CheckConnection` → `c.Impl().Xxx` → unwrap pattern; management calls use `AuthorizedContext` with the access token):

```go
func (c Client) SetupMFA(accessToken string) (secret, otpauthURL, qrPNGBase64 string, err error)
func (c Client) EnableMFA(accessToken, code string) (backupCodes []string, err error)
func (c Client) DisableMFA(accessToken, password string) error
func (c Client) GetMFAStatus(accessToken string) (enabled bool, err error)
func (c Client) VerifyMFA(mfaToken, code string, rememberMe bool, info ...ClientInfo) (accessToken, refreshToken string, err error)
func (c Client) GenerateBackupCodes(accessToken, password string) (backupCodes []string, err error)
```

`VerifyMFA` forwards `ClientInfo.IP` like `Login`/`Refresh` (the gateway resolves the IP once, so the issued refresh token is bound to it). Update `grpcclients/authclient/types.go` if it mirrors method signatures (verify; if not, skip).

---

## 2. `swayrider-api/internal/handlers/auth.go`

**`AuthClient` interface** (the gateway's narrow seam): update `Login` and add the six methods above.

**Handlers** (mirroring the existing `Login`/`ChangePassword` shapes — management handlers read the JWT via `security.GetJwt(r.Context())` like `ChangePassword` does):

- **`Login`**: after `h.client.Login(...)`:
  ```go
  if mfaRequired {
      // No cookies — the user must complete the second factor first.
      writeJSON(w, http.StatusOK, map[string]any{
          "mfa_required": true,
          "mfa_token":    mfaToken,
      })
      return
  }
  ```
  existing behavior otherwise (set cookies + token body).
- **`MfaSetup`** — `POST /api/v1/auth/mfa/setup` → `{secret, otpauth_url, qr_png_base64}`.
- **`MfaEnable`** — `POST /api/v1/auth/mfa/enable` → `{backup_codes: [...]}`.
- **`MfaDisable`** — `POST /api/v1/auth/mfa/disable` → 204.
- **`MfaStatus`** — `GET /api/v1/auth/mfa/status` → `{enabled: bool}`.
- **`MfaVerify`** — `POST /api/v1/auth/mfa/verify` → on success `setAuthCookies` + `{access_token, refresh_token}` (exactly like `Login`/`Refresh`).
- **`MfaBackupCodes`** — `POST /api/v1/auth/mfa/backup-codes` → `{backup_codes: [...]}`.

**`errBody`**: no new `reason` fields needed — the app treats any non-200 from verify as "code incorrect or expired" (see Plan 4). The existing prefix machinery stays as-is; no change required unless Plan 5 wants a distinct message (then add a `mfa_required`/`invalid_mfa_code` reason — decide there, keep it out of this plan).

## 3. `swayrider-api/internal/server/routes.go`

```go
mux.HandleFunc("POST /api/v1/auth/mfa/setup", auth.MfaSetup)
mux.HandleFunc("POST /api/v1/auth/mfa/enable", auth.MfaEnable)
mux.HandleFunc("POST /api/v1/auth/mfa/disable", auth.MfaDisable)
mux.HandleFunc("GET  /api/v1/auth/mfa/status", auth.MfaStatus)
mux.HandleFunc("POST /api/v1/auth/mfa/verify", auth.MfaVerify)
mux.HandleFunc("POST /api/v1/auth/mfa/backup-codes", auth.MfaBackupCodes)
```

## 4. Rate limiting — `swayrider-api/internal/middleware/ratelimit.go`

Add `POST /api/v1/auth/mfa/verify` to the **`auth`** group (alongside `/login`, `/register`, `/request-password-reset`, `/verify-email`) — it is the TOTP brute-force surface. Management endpoints (`setup`/`enable`/`disable`/`backup-codes`) go in the default `api` group.

## 5. Docs

- `swayrider-api/README.md` + `API.md`: new endpoints table rows, login MFA branch (200 with `mfa_required`, no cookies).
- `swayrider-api/api/openapi.yaml`: six new paths + `LoginResponse` shape note (add `mfa_required`/`mfa_token` to the login response schema).

## 6. Tests

- `internal/handlers/auth_test.go` (or a new `mfa_test.go`): stub `AuthClient` — login with `mfaRequired=true` → 200, `mfa_required: true`, `mfa_token` set, **no** cookies; `mfaRequired=false` → existing behavior; verify success → cookies + tokens; verify failure → 401 sanitized body.
- `internal/middleware/ratelimit_test.go`: `/api/v1/auth/mfa/verify` classified `auth`; `mfa/setup` etc. classified `api`.
- `errors_test.go`: no change expected (no new reasons) — confirm existing cases still pass.

## Definition of done (Plan 3)

- `grpcclients` builds; **all** `Login` callers migrated (grep proves none left).
- Gateway routes registered; handler unit tests green; rate-limit classification tested.
- `curl` sanity: login on an MFA-enabled account returns `mfa_required`, verify returns tokens (manual, against local stack).
- Docs updated (`README.md`, `API.md`, `openapi.yaml`).
- No app changes yet (Plan 4+).
