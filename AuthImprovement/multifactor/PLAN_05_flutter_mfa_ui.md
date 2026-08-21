# PLAN 05 — Flutter MFA management UI

**Layer:** `swayriderapp` UI (profile, setup flow, login verify step), localization.
**Depends on:** Plan 4 (repository/API methods exist).

---

## 1. Routes — `lib/routing/routes.dart`

```dart
static const mfaVerify = '/mfa-verify';   // login second-factor step
static const mfaSetup  = '/mfa-setup';    // enrollment flow (intra-screen steps)
```

- `mfaVerify` takes the pending `mfaToken` as a route extra (go_router) — the verify viewmodel is constructed with it.
- `mfaSetup` is reachable from Profile; it is NOT a public route (requires an authenticated session — consistent with `changePassword`).

## 2. Domain model — `lib/domain/models/mfa/mfa_setup_info.dart`

```dart
class MfaSetupInfo {
  final String secret;        // base32, grouped for display in the UI
  final String otpauthUrl;
  final String qrPngBase64;   // server-rendered QR, decoded with Image.memory
}
```

`AuthRepositoryRemote.setupMfa` maps `SetupMFAResponse` → `MfaSetupInfo`.

## 3. Profile section — `lib/ui/profile/widgets/profile_screen.dart`

Add a second card/row below the Change Password button: **Two-factor authentication** with status + action:

- `MfaProfileViewModel` (new, `ui/profile/view_models/mfa_profile_viewmodel.dart`): on load → `getMfaStatus()`; getters `enabled`, `loading`, `error`.
- Enabled → row "Two-factor authentication: **On**" + `Disable 2FA` button → password dialog → `disableMfa(password)` → refresh status + snackbar.
- Disabled → `Enable 2FA` button → `context.push(Routes.mfaSetup)`; on return `true`, refresh status.
- Follow the existing `_onChangePasswordPressed` push-and-wait-for-result pattern (`context.push<bool>`).

## 4. Setup flow — `lib/ui/mfa_setup/` (viewmodel + screen)

One route, internal step state (secret/token stay in the viewmodel — no fragile cross-route passing):

`MfaSetupViewModel`:
```dart
// step 1 → setupMfa()
late final Command1<void, void> startSetup;   // on Ok stores MfaSetupInfo
// step 3 → enableMfa(code)
late final Command1<void, String> enable;      // on Ok stores backupCodes
MfaSetupInfo? get setupInfo;
List<String>? get backupCodes;
bool get invalidCode => enable.result is Error;
```

Screen steps (`MfaSetupScreen`, stateful):
1. **Intro** — explains what 2FA is, that a third-party authenticator app is required, and that a manual key will be shown (QR is only for a second device). Button: *Start setup* → `startSetup`.
2. **Key + QR** — shows the base32 secret grouped as `ABCD EFGH IJKL MNOP QRST UVWX YZ12 3456` (format helper: 4-char groups, uppercase, spaces), a *Copy key* button (`Clipboard.setData` + snackbar), the QR PNG via `Image.memory(base64Decode(info.qrPngBase64))` with a caption "Scan with a second device, or enter the key manually", and a *I've added the key* button → step 3.
3. **Code entry** — 6-digit `AppTextField` (numeric, maxLength 6) + *Verify* → `enable`; on `invalidCode` show `localization.mfaInvalidCode` (same precedence pattern as other screens: exclude from generic error).
4. **Backup codes** — success screen listing `backupCodes` (monospace, one per line), warning that they are shown once, *I've saved them* → `context.pop(true)` (profile refreshes status).

## 5. Verify step (login) — `lib/ui/mfa_verify/` (viewmodel + screen)

`MfaVerifyViewModel`:
```dart
MfaVerifyViewModel({required AuthRepository authRepository, required String mfaToken});
late final Command1<void, String> verify;   // verifyMfa(mfaToken, code)
bool get invalidCode => verify.result is Error;
```

Screen (`MfaVerifyScreen`): branded scaffold consistent with `LoginScreen` — title (`localization.mfaVerifyTitle`), 6-digit code field, *Verify* button, inline `invalidCode` error, and a *Use a backup code* toggle that switches the field label but calls the same `verify` command (server accepts TOTP or backup codes in the same field). On `verify` Ok → `context.go(Routes.home)` (root replacement — matches how login completes). Token arrives via route extra (see §1).

## 6. Localization — `lib/ui/core/localization/`

Base getters in `applocalization.dart` + strings in `applocalization_en.dart` / `applocalization_nl.dart` (Dutch translations). Keys (representative; final set pinned during implementation):

```
twoFactorAuthentication, mfaEnabled, mfaDisabled,
enableTwoFactor, disableTwoFactor,
mfaSetupTitle, mfaSetupIntro, startSetup,
mfaSecretKey, copyKey, keyCopied,
mfaQrHint, mfaAddedKey,
mfaCodeLabel, verify, mfaInvalidCode,
mfaBackupCodesTitle, mfaBackupCodesIntro, mfaBackupCodesShownOnce, mfaBackupCodesSaved,
mfaVerifyTitle, mfaUseBackupCode,
mfaDisablePasswordPrompt, mfaDisableSuccess, mfaEnableSuccess
```

## 7. Tests

Per the repo convention (no screen-level widget tests; viewmodels are the seam):

- `test/ui/mfa_setup/view_models/mfa_setup_viewmodel_test.dart` — `startSetup` Ok stores `setupInfo`; `enable` Ok stores `backupCodes`; `enable` Error → `invalidCode` true.
- `test/ui/mfa_verify/view_models/mfa_verify_viewmodel_test.dart` — verify Ok/Error; token passed through to the repository call (mocktail `AuthRepository` mock).
- `test/ui/profile/view_models/mfa_profile_viewmodel_test.dart` — status load, disable result.
- Localization: follow whatever the existing localization test convention is (check `test/ui/core/localization/`; add key-completeness assertions if one exists).

## Definition of done (Plan 5)

- `flutter analyze` + `flutter test` green.
- Full manual pass on a device: enable 2FA via manual key (authenticator on same phone), login with TOTP, login with a backup code, disable 2FA, regenerate backup codes. QR enrollment with a second device.
- En/nl strings in place; profile shows live status.
- Phase-3 doc §1 marked implemented; this plan's completion closes out the MFA feature.
