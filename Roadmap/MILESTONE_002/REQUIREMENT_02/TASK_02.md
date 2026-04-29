# Task 02 - Improve Error Handling with Proper gRPC Status Codes

**Status**: closed

## Scope

- `backend/services/routerservice/internal/server/`
- `backend/services/routerservice/internal/logic/`

## Description

Enhance error handling to return well-defined gRPC status codes:

1. **Valhalla backend unreachable** — When the Valhalla client returns a network/connection error, return gRPC status `UNAVAILABLE` instead of `Internal`

2. **Location outside covered regions** — When `ErrLocationOutsideOfKnownRegions` is detected, return gRPC status `NOT_FOUND` (per requirement) instead of `InvalidArgument`

3. **No route found** — Ensure `ErrNoRouteFound` returns appropriate gRPC status

## Instructions

### Error Handling Updates

**File: `backend/services/routerservice/internal/server/status.go`**

1. Add new error types or import from valhalla client:
   - Define or import `ErrValhallaUnavailable` or detect network errors
   - Update the `grpcStatus` function to map:
     - Valhalla connection errors → `codes.Unavailable`
     - `ErrLocationOutsideOfKnownRegions` → `codes.NotFound`
     - `ErrNoRouteFound` → `codes.NotFound`

**File: `backend/services/routerservice/internal/server/route.go`**

1. Wrap Valhalla client errors with proper classification:
   - When `vhClient.Route()` returns an error, check if it's a network/connection error
   - If network error, wrap with `ErrValhallaUnavailable` or similar
   - Ensure the error propagates to `grpcStatus()` which will convert to `UNAVAILABLE`

**File: `backend/services/routerservice/internal/logic/errors.go`**

1. Add new error variable if needed:
   ```go
   var ErrValhallaUnavailable = errors.New("Valhalla backend unavailable")
   ```

## Test Scenarios

1. **Valhalla unreachable** — Mock/simulate Valhalla client to return connection error; verify response has gRPC status `UNAVAILABLE`
2. **Location outside regions** — Mock region client to indicate location outside known regions; verify response has gRPC status `NOT_FOUND`
3. **No route found** — When no viable route exists between locations; verify response has gRPC status `NOT_FOUND`
4. **Other errors** — Verify unexpected errors still return `Internal` (not leaked to client)

## Acceptance Criteria

- [ ] Request with origin or destination outside all covered regions returns gRPC `NOT_FOUND`
- [ ] Request when Valhalla backend is unreachable returns gRPC `UNAVAILABLE`
- [ ] Request with no route found returns gRPC `NOT_FOUND`
- [ ] Internal errors are not leaked to clients (return generic message)
- [ ] All existing unit tests continue to pass
- [ ] New error case unit tests pass
- [ ] Service compiles without errors
