# PLAN 04 — Flutter API client + repository + login MFA step

**Layer:** `swayriderapp/lib/data/services/api`, `lib/data/repositories/auth`, `lib/ui/login`.
**Depends on:** Plan 3 (gateway endpoints exist).

---

## 1. Models — `swayriderapp/lib/data/services/api/model/auth/auth.dart`

- `LoginResponse` gains:
  ```dart
  final bool mfaRequired;
  final String? mfaToken;
  // fromJson: json['mfa_required'] == true, json['mfa_token'] as String?
  ```
- New models (freezed/plain classes per existing style): `SetupMFAResponse` (`secret`, `otpauthUrl`, `qrPngBase64`), `EnableMFAResponse` (`backupCodes`), `MfaStatusResponse` (`enabled`), `VerifyMFAResponse` (`accessToken`, `refreshToken`).

## 2. `AuthApiClient` — new methods + login field parsing

`login()` already returns `Result<LoginResponse>` — parsing the new fields is the only change there.

```dart
Future<Result<SetupMFAResponse>> setupMfa();                    // POST /mfa/setup, auth header
Future<Result<EnableMFAResponse>> enableMfa(String code);       // POST /mfa/enable
Future<Result<void>> disableMfa(String password);               // POST /mfa/disable
Future<Result<MfaStatusResponse>> getMfaStatus();               // GET  /mfa/status
Future<Result<VerifyMFAResponse>> verifyMfa(String mfaToken, String code); // POST /mfa/verify
Future<Result<EnableMFAResponse>> generateBackupCodes(String password);    // POST /mfa/backup-codes
```

Conventions to follow: management calls attach `_authHeader` (like `changePassword`); verify is unauthenticated (like `login`). Errors → `Result.error(HttpException(...))`; 401 on management calls → `UnauthorizedException()` (so `withAuthRetry` works).

## 3. Repository — `auth_repository.dart` + `auth_repository_remote.dart`

**`login()` return type changes** — this is the app-side contract break:

```dart
sealed class LoginOutcome {}
class LoginSuccess extends LoginOutcome {}
class LoginMfaRequired extends LoginOutcome {
  final String mfaToken;
  LoginMfaRequired(this.mfaToken);
}

Future<Result<LoginOutcome>> login({required String email, required String password, bool rememberMe = false});
```

- `Ok(LoginSuccess)` → `_saveTokens` + `notifyListeners` (existing behavior).
- `Ok(LoginMfaRequired(token))` → do **not** save tokens; return the outcome so the UI routes to the code-entry screen.
- `Error` → unchanged.

New repository methods (all `withAuthRetry`-wrapped except `verifyMfa`, which is unauthenticated):

```dart
Future<Result<void>> verifyMfa({required String mfaToken, required String code}); // saves tokens on Ok
Future<Result<MfaSetupInfo>> setupMfa();                                          // domain model, see Plan 5
Future<Result<List<String>>> enableMfa({required String code});
Future<Result<void>> disableMfa({required String password});
Future<Result<bool>> getMfaStatus();
Future<Result<List<String>>> generateBackupCodes({required String password});
```

`verifyMfa` maps `Ok(VerifyMFAResponse)` → `_saveTokens` + `notifyListeners` (same as `login`).

## 4. `LoginViewModel` — `ui/login/view_models/login_viewmodel.dart`

```dart
// login command stays (its Result now carries LoginOutcome)
LoginOutcome? get loginOutcome => (login.result as Ok<LoginOutcome>?)?.value;
bool get mfaRequired => loginOutcome is LoginMfaRequired;
String? get mfaToken => (loginOutcome as LoginMfaRequired?)?.mfaToken;

// second-factor completion (the verify screen drives this)
late final Command1<void, (String mfaToken, String code)> verifyMfa;
```

## 5. Login screen — `ui/login/widgets/login_screen.dart`

On `login` completion inside the existing `ListenableBuilder`:

- `LoginSuccess` → `context.go(Routes.home)` (existing behavior).
- `LoginMfaRequired` → navigate to the code-entry route with the token (route + screen land in Plan 5; here only the navigation hook and a `Routes.mfaVerify` constant are added).
- `Error` → existing `invalidLogin` message.

## 6. Tests

- `test/data/services/api/auth_api_client_test.dart`: login parses `mfa_required`/`mfa_token`; each new method maps 200 bodies / 401 / error bodies (existing `HttpClient`-factory mock pattern).
- `test/data/repositories/auth/auth_repository_test.dart` (if the repo has one — check; otherwise add): `login` Ok+`mfa_required` → `LoginMfaRequired` outcome and **no** token save (assert storage untouched); Ok without → `LoginSuccess` + tokens saved; `verifyMfa` Ok → tokens saved.
- `test/ui/login/view_models/login_viewmodel_test.dart`: outcome getters; verify command result passthrough.

## Definition of done (Plan 4)

- `flutter analyze` + `flutter test` green.
- App can complete a full MFA login end-to-end against the local stack (login → code entry screen → home) once Plan 5 renders the screen; the data path is proven by tests.
- No MFA *management* UI yet (Plan 5).
