# TASK_003 — Documentation Updates

**Status**: Done

## Overview

Update documentation files to remove references to the web authentication portal.

## Repository

- **Repo**: swayrider
- **Subfolder**: root, `Docs/`
- **Tech**: Markdown

## Scope: Modify

- `README.md` — Remove web portal references
- `BUILDING.md` — Remove web portal build instructions (if exists)
- `Docs/` — Remove web portal from architecture docs (if referenced)

## Technical Specification

### Files to Check and Update

1. `README.md` — Search for "web", "auth portal", "authportal", "frontend"
2. `BUILDING.md` — Search for web build instructions
3. `Docs/` — Architecture diagrams or references
4. `AGENTS.md` / `CLAUDE.md` — Any web-specific agent rules

### Search Commands

```bash
grep -rn "web" README.md BUILDING.md Docs/
grep -rn "authportal\|auth portal\|auth-portal" README.md BUILDING.md Docs/
grep -rn "frontend-libs\|frontend-utils" README.md BUILDING.md Docs/
```

## Dependencies

- TASK_001 (code must be removed first)

## Acceptance Criteria

- [ ] README.md has no web portal references
- [ ] BUILDING.md has no web portal build instructions
- [ ] Docs/ has no web portal architecture references

## Testing Notes

- Manual review of updated documentation
- Verify links still work after removal
