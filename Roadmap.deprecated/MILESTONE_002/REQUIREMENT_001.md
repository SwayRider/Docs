# REQUIREMENT_001 — Remove Web Auth Portal

## Overview

Remove the deprecated React web authentication portal from the repository, including all associated code, configuration, and documentation references.

## Context

- **Components**: Web Frontend, Infrastructure
- **Priority**: Medium
- **Status**: Done

## Background

The React web auth portal was developed as an alternative authentication interface. With the mobile-first strategy, the web portal is no longer needed and adds unnecessary complexity to the codebase.

## Requirements

### Code Removal
- Remove `web/` directory completely
- Remove any shared code that only served the web portal
- Clean up imports and references

### Infrastructure Cleanup
- Remove web portal from Docker compose configurations
- Update Traefik routing to remove web portal routes
- Remove web portal build targets from Makefile
- Clean up any web-specific environment variables

### Documentation Updates
- Update README.md to remove web portal references
- Update BUILDING.md to remove web portal build instructions
- Remove web portal from architecture diagrams
- Update any deployment documentation

### Verification
- Ensure no broken imports or references remain
- Verify build succeeds without web portal
- Confirm CI/CD pipeline passes

## Acceptance Criteria

1. `web/` directory does not exist in repository
2. No Go or configuration files reference the web portal
3. Docker compose files do not include web portal services
4. Makefile does not include web portal targets
5. Documentation does not reference web portal
6. All builds pass successfully
7. CI/CD pipeline passes

## Affected Files

### Directories to Remove
- `web/` — Complete directory

### Files to Modify
- `Makefile` — Remove web-related targets
- `README.md` — Remove web portal references
- `BUILDING.md` — Remove web portal build instructions
- `infra/dev/` — Docker compose updates
- Traefik configuration files

### Files to Verify
- Any imports or references to web portal code
- Environment variable configurations
- CI/CD pipeline configurations

## Out of Scope

- Removal of authservice web endpoints (still needed for email verification pages)
- Changes to mobile app
- Changes to backend services

## Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| Hidden dependencies on web code | Low | Thorough search before removal |
| Active users of web portal | Low | Portal is deprecated |

## Timeline

| Task | Duration |
|------|----------|
| Code removal | 0.5 days |
| Infrastructure cleanup | 0.5 days |
| Documentation updates | 0.5 days |
| Verification | 0.5 days |
| **Total** | **2 days** |
