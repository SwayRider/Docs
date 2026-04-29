# TASK_002 — Infrastructure Cleanup

**Status**: Done

## Overview

Remove web portal references from the Makefile and verify no other infrastructure configs reference the web portal.

## Repository

- **Repo**: swayrider
- **Subfolder**: `Makefile`, `infra/`
- **Tech**: Make, Traefik, Docker

## Scope: Modify

- `Makefile` — Remove `web-containers:` target (line 121)

## Technical Specification

### Makefile Changes

Remove the empty `web-containers:` target at line 121:
```makefile
web-containers:
```

### Traefik Config (No Changes Needed)

`infra/dev/layer-00/traefik/traefik.yml` contains:
- `web:` — Standard HTTP entrypoint (port 80)
- `websecure:` — Standard HTTPS entrypoint (port 443)

These are **not** web portal references — they are Traefik's built-in entrypoint names. Do NOT remove.

### Docker Compose (No Changes Needed)

No docker-compose files reference the web portal. Verified by searching for `authportal`, `web/auth` in `infra/`.

## Dependencies

- TASK_001 (code must be removed first)

## Acceptance Criteria

- [ ] `web-containers:` target removed from Makefile
- [ ] No Docker compose services reference web portal
- [ ] Traefik entrypoints unchanged (web/websecure are standard)

## Testing Notes

- `make --dry-run` should not list web-containers
- `docker compose config` should not include web services
