# REQUIREMENT_006 — Remove MinIO from Services

## Overview

Remove MinIO dependency from MailService and RegionService, replacing object store reads with direct filesystem reads.

## Context

- **Original Requirement**: 26_006
- **Components**: MailService, RegionService
- **Priority**: High
- **Status**: Done

## Background

MinIO is no longer available as an open-source platform. Services currently using it for storage should migrate to filesystem-based storage.

## Requirements

### MailService
- Move mail templates from `backend/templates/mail` to `assets/mail/templates`
- Remove MinIO dependency, flags, and environment variables
- Add configuration to point to mail templates directory
- Load templates from configured directory

### RegionService
- Replace MinIO client with direct filesystem reads
- Single directory flag (`--geodata-dir`) points to geodata root
- Existing `--tag` flag continues to work as subdirectory selector
- Remove all MinIO configuration flags

## Acceptance Criteria

1. MailService loads templates from local filesystem
2. MailService has no MinIO imports or configuration
3. RegionService loads geodata from local filesystem
4. RegionService has no MinIO imports or configuration
5. All existing functionality preserved
6. CI pipeline passes with no MinIO references

## Affected Files

### Backend
- `backend/services/mailservice/` — Template loading refactor
- `backend/services/regionservice/` — Geodata loading refactor
- `backend/services/regionservice/internal/objectstore/` → rename to `geodata`
- `backend/swlib/app/` — Remove MinIO wiring if no longer needed

### Configuration
- Environment variables updated
- Docker compose configurations updated

## New Configuration

| Flag | Env Var | Description |
|------|---------|-------------|
| `--template-dir` | `TEMPLATE_DIR` | MailService: path to mail templates |
| `--geodata-dir` | `GEODATA_DIR` | RegionService: root directory for geodata |

## Out of Scope

- Data pipeline MinIO upload logic (separate ticket)
- Changes to other services
- Format changes to data files
