# Task 01 - Secure RouterService Route Endpoint with JWT Authentication

**Status**: completed

## Description

The RouterService `Route` gRPC endpoint is currently public and accepts unauthenticated
requests. This task wires JWT authentication into the RouterService so that `Route`
requires a valid, email-verified JWT while `Ping` remains public.

The implementation follows the established auth interceptor pattern used by other
services in the monorepo (e.g. SearchService).

### Instructions

**File: `backend/services/routerservice/internal/server/server.go`**

- In the `init()` function, remove the `security.PublicEndpoint` registration for
  the `Route` method.
- Keep `security.PublicEndpoint` for `Ping`.
- The zero-value `EndpointProfile` (secure, requires valid JWT + email verified) will
  then apply automatically to `Route`.

**File: `backend/services/routerservice/cmd/routerservice/main.go`**

- Declare `kc := &keyCache{}` before `app.New()`.
- Wire the `authclient` as a service client using `.WithServiceClients(...)`.
- Add `.WithAppData("keyCache", kc)` and `.WithBackgroundRoutines(keyFetcherRoutine)`
  to the app builder chain.
- `keyFetcherRoutine` starts `authclient.PublicKeyFetcher` in a goroutine (mirrors
  the pattern in SearchService / other secured services).
- Replace `app.NoInterceptor` with `app.AuthInterceptor|app.ClientInfoInterceptor`
  in the `app.NewGrpcConfig(...)` call.

Follow the exact keyCache / keyFetcherRoutine pattern already present in other
services (e.g. `backend/services/searchservice/cmd/searchservice/main.go`).

### Test Scenarios

1. **Unauthenticated Route call** — send a `Route` RPC without a JWT; expect
   gRPC status `Unauthenticated`.
2. **JWT present, email not verified** — send a `Route` RPC with a JWT whose
   email-verified claim is false; expect gRPC status `PermissionDenied`.
3. **Valid JWT** — send a `Route` RPC with a valid, email-verified JWT; expect
   a successful route response.
4. **Ping without JWT** — call `Ping` without any token; expect a successful
   response (public endpoint unchanged).

## Acceptance Criteria

- [*] `Route` endpoint requires a valid, email-verified JWT; unauthenticated
      calls are rejected with `Unauthenticated`.
- [*] `Ping` endpoint remains public and accepts calls without a token.
- [*] `main.go` wires `kc`, `keyFetcherRoutine`, `authclient` service client,
      and uses `app.AuthInterceptor|app.ClientInfoInterceptor`.
- [*] `server.go` no longer registers `Route` as a public endpoint.
- [*] All existing unit tests in `backend/services/routerservice/...` continue
      to pass.
- [*] The service compiles without errors (`go build ./...` in `backend/`).
