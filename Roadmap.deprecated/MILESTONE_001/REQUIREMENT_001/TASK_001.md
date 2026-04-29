# TASK_001 — TilesService Style Serving

**Status**: Done

## Overview

Implement filesystem-based map style loading in TilesService.

## Context

- **Requirement**: REQUIREMENT_001
- **Component**: TilesService
- **Estimated Effort**: 1 day

## Implementation

### Configuration
Add new configuration field for styles directory:
- Flag: `--styles-dir`
- Environment: `STYLES_DIR`
- Default: `./styles`

### Loading Logic
1. Read directory at startup
2. Parse style JSON files
3. Build style registry (name → content)
4. Validate JSON structure
5. Support hot-reload (optional)

### Style Naming
- Default: `light.json`, `dark.json`
- Custom: `{name}-light.json`, `{name}-dark.json`
- Extension stripped for API

## Acceptance Criteria

- [ ] Styles directory configurable
- [ ] All JSON files in directory loaded
- [ ] Naming convention enforced
- [ ] Invalid JSON logged and skipped
- [ ] Style registry built at startup

## Files to Modify

- `backend/services/tilesservice/internal/styles/` — New package
- `backend/services/tilesservice/cmd/main.go` — Add flag
