# REQUIREMENT_006 — Tasks Index

## Tasks

| ID | Name | Status | Description |
|----|------|--------|-------------|
| TASK_001 | MailService MinIO Removal | Done | Remove MinIO from MailService, use filesystem |
| TASK_002 | RegionService MinIO Removal | Done | Remove MinIO from RegionService, use filesystem |
| TASK_003 | Shared MinIO Cleanup | Done | Remove MinIO from swlib if no longer needed |

## Task Dependencies

```
TASK_001 (MailService) ──► TASK_002 (RegionService) ──► TASK_003 (swlib cleanup)
```

## Acceptance Criteria Summary

- [x] MailService loads templates from filesystem
- [x] RegionService loads geodata from filesystem
- [x] No MinIO imports remain in either service
- [x] swlib MinIO code removed if no longer needed
