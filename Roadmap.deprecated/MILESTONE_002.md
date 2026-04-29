# MILESTONE_002 — Infrastructure Cleanup

## Overview

Remove the deprecated React web authentication portal and clean up related infrastructure, simplifying the codebase and reducing maintenance burden.

## Scope

- **Phase**: Pre-MVP
- **Priority**: Medium
- **Dependencies**: None
- **Blocks**: MILESTONE_003 (minor)

## Background

The React web auth portal is deprecated as part of the mobile-first strategy. The web interface is no longer in scope and should be removed to:
- Reduce codebase complexity
- Eliminate maintenance overhead
- Simplify deployment
- Focus resources on mobile applications

## Requirements

| ID | Name | Components | Status |
|----|------|------------|--------|
| [REQUIREMENT_001](./MILESTONE_002/REQUIREMENT_001.md) | Remove Web Auth Portal | Web Frontend, Infrastructure | Planned |

## Affected Components

### Web Frontend
- `web/` directory — Complete removal
- React application code
- Associated dependencies and configuration

### Infrastructure
- Docker compose configurations
- Traefik routing rules
- Build scripts referencing web portal

### Documentation
- References to web portal in README, BUILDING.md, etc.

## Success Criteria

1. `web/` directory removed from repository
2. No references to web portal in infrastructure configs
3. Build scripts updated to exclude web portal
4. Documentation updated
5. No broken references or imports
6. CI/CD pipelines pass without web portal

## Timeline Estimate

| Task | Estimated Effort |
|------|------------------|
| Code removal | 0.5 days |
| Infrastructure cleanup | 0.5 days |
| Documentation updates | 0.5 days |
| Testing and verification | 0.5 days |
| **Total** | **2 days** |

## Risks

- **Low Risk**: Web portal is deprecated and not actively used
- **Mitigation**: Verify no active users before removal

## Rollback Plan

- Git history preserves all code
- Can restore from previous commit if needed
