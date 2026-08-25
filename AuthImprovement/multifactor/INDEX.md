# Multi-Factor Authentication — Implementation Plans

> **Status: all five plans implemented (2026-08-21).** TOTP core + db (1), authservice endpoints + login flow (2), gateway + authclient (3), Flutter API/repository/login step (4), and Flutter management UI (5) are complete end-to-end.

Sequential implementation plans for [AUTH_IMPROVEMENTS_PHASE_03 §1](../AUTH_IMPROVEMENTS_PHASE_03.md) (TOTP MFA).

The feature spans five layers, each an independently committable unit. Implement strictly in order — each plan builds on the previous one's schema/contracts, and Plan 3 breaks the `authclient.Login` signature (all consumers must move in the same step).

## Decisions (agreed 2026-08-21)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Authenticator app | Standard third-party TOTP app (Google Authenticator, Authy, 1Password, …) — RFC 6238 | No external service, no cost, works with any app |
| Enrollment UX | **Manual base32 key (primary) + server-generated QR PNG (secondary)** | A phone cannot scan its own screen, so QR is only usable with a second device/desktop; manual key is the same-device path. QR rendered server-side (`authservice`), returned as base64 PNG in the `SetupMFA` response |
| Manual key size | 20-byte secret → 32 base32 chars, grouped `ABCD EFGH IJKL MNOP QRST UVWX YZ12 3456` + copy button | Comparable to a Wi-Fi password; copy button covers paste-capable authenticators |
| MFA challenge token | **DB-backed single-use token** (SHA-256-hashed, TTL, attempt counter) — mirrors `reset_password_tokens` | Revocable, per-challenge brute-force limit, matches existing patterns |
| TOTP brute force | Per-user throttle scope (`mfa`) in `security_throttle` **in addition to** the per-challenge attempt counter | A password-phishing attacker can loop successful password logins, so login lockout alone does not bound TOTP guessing |
| TOTP secret at rest | **AES-256-GCM via `swlib/encryption` / `ENCRYPTION_MASTER_KEY`** (same `KeyRing` as `jwt_keys.private_key`) | The secret IS the second factor; plaintext TEXT (as sketched in the phase doc) would leak it on DB dump |
| Backup codes | 10 × 8-char Crockford base32, Argon2id-hashed, single-use, shown once | Standard practice; regenerating invalidates old codes |
| MFA scope | Opt-in per user; login-time only (refresh keeps working after a completed MFA login) | Per the phase doc |
| QR library | `github.com/skip2/go-qrcode` (incumbent) or `github.com/yeqown/go-qrcode/v2` (actively maintained) — **verify at implementation time** (Plan 2) | Dependency lives in `authservice`, not `swlib` |

## Deviations from the phase-doc sketch

- **`swlib/totp` stays stdlib-only.** The sketch's `GenerateQRCodeURL` becomes `GenerateOTPAuthURL` (returns the `otpauth://` string). PNG rendering lives in `authservice` (needs a third-party lib, so it must not drag a dependency into the shared `swlib` module). See [PLAN_01](PLAN_01_totp_and_db.md) and [PLAN_02](PLAN_02_authservice_endpoints.md).
- **Challenge token is DB-backed** (not a short-lived JWT) — see decisions above.
- **`user_mfa.secret` is encrypted at rest** with a `secret_key_id` fingerprint column (mirrors `jwt_keys`), enabling `ENCRYPTION_MASTER_KEY_PREVIOUS`-based rotation.
- **New `mfa` throttle scope** in `security_throttle` (the sketch had no MFA brute-force protection beyond the login lockout).
- **`MFA_ENABLED=false`** fails closed for management endpoints (returns `FailedPrecondition`) and bypasses the MFA step in login.

## Plan sequence

| # | Plan | Layer | Depends on |
|---|------|-------|------------|
| 1 | [PLAN_01 — TOTP core + config + schema + db layer](PLAN_01_totp_and_db.md) | `swlib/totp`, `authservice` config/migrations/db | — |
| 2 | [PLAN_02 — protos + authservice endpoints + login flow](PLAN_02_authservice_endpoints.md) | `protos`, `authservice/internal/server` | 1 |
| 3 | [PLAN_03 — gateway + authclient wrapper](PLAN_03_gateway_and_client.md) | `grpcclients/authclient`, `swayrider-api` | 2 |
| 4 | [PLAN_04 — Flutter API client + repository + login step](PLAN_04_flutter_api_and_login.md) | `swayriderapp` data layer, login flow | 3 |
| 5 | [PLAN_05 — Flutter MFA management UI](PLAN_05_flutter_mfa_ui.md) | `swayriderapp` profile/setup/verify screens | 4 |

## Cross-cutting requirements (apply in every plan that touches a layer)

- **Pre-commit**: `go test ./...`, `go vet ./...`, `golangci-lint run ./...` from each touched Go repo; `flutter analyze` + `flutter test` for the app.
- **Docs**: `authservice/README.md`, `authservice/env.example`, `swayrider-api/README.md`, `swayrider-api/API.md`, `swayrider-api/api/openapi.yaml`, `grpcclients/README.md` (if it documents methods), `Docs/AuthImprovement/AUTH_IMPROVEMENTS_PHASE_03.md` §1 status.
- **Env vars** (registered in `authservice/cmd/authservice/main.go`, mirrored in `env.example`):

```bash
MFA_ENABLED=true                    # global switch; false bypasses MFA in login and fails closed on MFA management
MFA_CODE_LENGTH=6                   # TOTP digits
MFA_TIME_STEP=30                    # seconds per TOTP window
MFA_GRACE_PERIOD=1                  # accept the previous/next window
MFA_BACKUP_CODES=10                 # number of backup codes issued
MFA_CHALLENGE_TTL_SECS=300          # lifetime of the pending-login challenge token
MFA_CHALLENGE_MAX_ATTEMPTS=5        # TOTP/backup-code guesses per challenge before it is invalidated
MFA_LOCKOUT_THRESHOLD=5             # failed MFA verifications before the user's MFA scope locks
MFA_LOCKOUT_WINDOW_SECS=900         # sliding window for the MFA lockout counter
MFA_LOCKOUT_DURATION_SECS=900       # how long the MFA scope stays locked
```

## Testing strategy (across plans)

- **Unit**: RFC 6238 vectors (`swlib/totp`); backup-code format/uniqueness; challenge TTL/attempts; encrypt/decrypt round-trip.
- **Server (mockDB)**: setup/enable/disable/status/verify/backup-code endpoint behavior; login branch (MFA on/off/global-off); throttle interplay; audit events; single-use enforcement for backup codes and challenges.
- **Gateway**: login MFA branch (no cookies), verify flow (cookies), route registration, rate-limit classification, `errBody` behavior.
- **Flutter**: `AuthApiClient` mapping, repository `LoginOutcome`, viewmodel getters (per the repo's no-screen-widget-test convention).
- **Manual**: end-to-end enrollment with a real authenticator app on a phone (manual key entry), second-device QR enrollment, login with TOTP, login with a backup code, disable.
