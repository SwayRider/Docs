# Requirement 01 - RouterService Authentication & Security

**Status**: Closed

**Scope**:
  - `backend/services/routerservice/internal/server/server.go`
  - `backend/services/routerservice/cmd/routerservice/main.go`

## Goal

Secure the RouterService Route endpoint behind JWT authentication, matching the same security pattern used by SearchService. The Route endpoint must require a valid, email-verified JWT token.

## Description

Currently the RouterService registers its `Route` endpoint as a `PublicEndpoint`, meaning it accepts unauthenticated requests. This must be changed so that `Route` uses the zero-value `EndpointProfile` (the secure default in the swlib security framework), which requires a valid JWT with email verified.

This mirrors the pattern already in place for SearchService:
- `Search` endpoint: zero-value EndpointProfile (no explicit registration = secure)
- `Ping` / health: `security.PublicEndpoint(...)`

The RouterService `main.go` must also be updated to wire up the JWT key cache and key fetcher background routine, following the Auth Interceptor Pattern documented in the project memory:
- `kc := &keyCache{}` before `app.New()`
- `.WithAppData("keyCache", kc).WithBackgroundRoutines(keyFetcherRoutine)`
- `app.NewGrpcConfig(app.AuthInterceptor|app.ClientInfoInterceptor, ...)`

The `Ping` / health endpoint remains public.

## Tasks

_Tasks to be defined._

## Dependencies

- MILESTONE_001 (Foundation Complete) — auth infrastructure (Keycloak, AuthService, JWT key distribution) must be operational

## Acceptance Criteria

- [ ] The `Route` gRPC endpoint is no longer registered as `PublicEndpoint`
- [ ] Calling `Route` without a JWT returns an `Unauthenticated` gRPC error
- [ ] Calling `Route` with an unverified email JWT returns a `PermissionDenied` gRPC error
- [ ] Calling `Route` with a valid, email-verified JWT succeeds (route is calculated and returned)
- [ ] The `Ping` / health endpoint remains publicly accessible without a token
- [ ] RouterService `main.go` wires the JWT key cache and fetcher background routine
- [ ] Existing RouterService unit tests continue to pass
