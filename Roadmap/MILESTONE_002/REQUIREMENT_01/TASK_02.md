# Task 02 - Update RouterService Bruno Collection to Support JWT Authentication

**Status**: completed

## Description

The RouterService Bruno collection does not support authenticated requests. This
task adds a Login folder (mirroring the SearchService pattern), extends the
environment files with authservice variables and an `access_token` placeholder,
and adds the `Authorization: Bearer {{access_token}}` header to the two Route
request files.

The `Ping` request is left unchanged (public endpoint).

### Instructions

**New file: `rest/RouterService/Login/folder.bru`**

Create a Bruno folder descriptor for the Login group. Mirror the structure used
in `rest/SearchService/Login/folder.bru`.

**New file: `rest/RouterService/Login/Login User [-].bru`**

Create a Bruno request that:
- Method: `POST`
- URL: `{{AUTHSERVICE_HOST}}:{{AUTHSERVICE_PORT}}/auth/v1/token`
- Body: JSON with `username`, `password`, `grant_type: "password"`
- Post-response script: extracts `access_token` and `refresh_token` from the
  response and stores them as environment variables.

Mirror the exact structure of `rest/SearchService/Login/Login User [-].bru`.

**Update: `rest/RouterService/environments/Local.bru`**

Add the following variables:
- `AUTHSERVICE_HOST` — local authservice host (e.g. `http://localhost`)
- `AUTHSERVICE_PORT` — local authservice port (e.g. `34001`)
- `access_token` — empty string placeholder

Mirror values used in `rest/SearchService/environments/Local.bru`.

**Update: `rest/RouterService/environments/SwayRider - Dev.bru`**

Add the same three variables with dev-environment values, mirroring
`rest/SearchService/environments/SwayRider - Dev.bru`.

**Update: `rest/RouterService/Route (1 region, a-b).bru`**

Add the HTTP header:
```
Authorization: Bearer {{access_token}}
```

**Update: `rest/RouterService/Route (2 regions, a-b).bru`**

Add the same HTTP header:
```
Authorization: Bearer {{access_token}}
```

### Test Scenarios

1. **Login request (Local env)** — execute `Login User [-]` in the Local
   environment; expect a 200 response and `access_token` / `refresh_token`
   set in environment variables.
2. **Route request with token** — after login, execute `Route (1 region, a-b)`
   in the Local environment; the `Authorization` header is present and the
   request succeeds (given a running RouterService with JWT auth enabled).
3. **Route request without token** — clear `access_token` in the environment
   and execute a Route request; the header is sent as `Bearer ` (empty), which
   the server rejects — confirms the header plumbing is in place.
4. **Ping request** — execute `Ping`; no `Authorization` header is present and
   the request succeeds without a token.
5. **Dev environment** — switch to `SwayRider - Dev` environment; verify
   `AUTHSERVICE_HOST` and `AUTHSERVICE_PORT` resolve to dev values.

## Acceptance Criteria

- [*] `rest/RouterService/Login/folder.bru` exists and is a valid Bruno folder
      descriptor.
- [*] `rest/RouterService/Login/Login User [-].bru` exists, POSTs to
      `{{AUTHSERVICE_HOST}}:{{AUTHSERVICE_PORT}}/auth/v1/token`, and stores
      `access_token` and `refresh_token` via a post-response script.
- [*] `rest/RouterService/environments/Local.bru` contains `AUTHSERVICE_HOST`,
      `AUTHSERVICE_PORT`, and `access_token` variables.
- [*] `rest/RouterService/environments/SwayRider - Dev.bru` contains the same
      three variables with dev-environment values.
- [*] `rest/RouterService/Route (1 region, a-b).bru` includes the
      `Authorization: Bearer {{access_token}}` header.
- [*] `rest/RouterService/Route (2 regions, a-b).bru` includes the same header.
- [*] `rest/RouterService/Ping.bru` (or equivalent) is unchanged — no
      Authorization header added.
- [*] All Bruno files are valid and parseable by the Bruno CLI (`bru run`
      dry-run passes).
