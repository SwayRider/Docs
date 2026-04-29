# TASK_001 — Code Removal

**Status**: Done

## Overview

Remove the `web/` directory entirely, containing the React auth portal and associated frontend packages.

## Repository

- **Repo**: swayrider
- **Subfolder**: `web/`
- **Tech**: React, TypeScript, Vite, Yarn

## Scope: Remove

- `web/authportal/` — React auth portal (login, register, password reset pages)
- `web/frontend-libs/` — Shared UI components (Button, Card, Input, Checkbox), auth context, JWT utilities
- `web/frontend-utils/` — Low-level auth utilities (JWT signature verification, decode)
- `web/package.json`, `web/yarn.lock`, `web/.yarn/`, `web/.yarnrc.yml` — Workspace root

## Technical Specification

### Directory Structure Being Removed

```
web/
  authportal/          — Main portal app (Vite + React)
    src/pages/         — Login, Register, ForgotPassword, ResetPassword, ChangePassword, VerifyToken
    src/forms/         — Form components
    src/utils/         — Password check utility
  frontend-libs/       — Shared component library
    src/components/    — Button, Card, Input, Checkbox
    src/context/       — AuthContext, ApiContext
    src/security/      — Security utilities
    src/jwt/           — JWT signature and decode
  frontend-utils/      — Low-level auth utilities
    src/auth/          — JWT signature verification, decode
```

### Pre-Removal Check

Search for any imports from `web/` in backend code:
```
grep -r "web/" backend/ --include="*.go"
grep -r "authportal" backend/ --include="*.go"
grep -r "frontend-libs" backend/ --include="*.go"
```

## Dependencies

- None (first task)

## Acceptance Criteria

- [ ] `web/` directory does not exist
- [ ] No orphaned references to web packages in other code

## Testing Notes

- After removal, run `grep -r "web/" . --include="*.go" --include="*.ts" --include="*.yml"` to find stragglers
