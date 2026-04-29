# REQUIREMENT_008 — Tasks Index

## Tasks

| ID | Name | Status | Description |
|----|------|--------|-------------|
| TASK_001 | SearchService Proto Definition | Done | Define gRPC service and messages |
| TASK_002 | SearchService Implementation | Done | Implement search logic with multi-phase flow |
| TASK_003 | Address Collapsing | Done | Implement street+locality grouping |
| TASK_004 | Ranking Logic | Done | Implement confidence+distance sorting |
| TASK_005 | Mobile App Refactor | Done | Replace Pelias calls with SearchService |

## Task Dependencies

```
TASK_001 (Proto) ──► TASK_002 (Implementation) ──┬─► TASK_003 (Collapsing)
                                                 ├─► TASK_004 (Ranking)
                                                 └─► TASK_005 (Mobile Refactor)
```

## Acceptance Criteria Summary

- [x] SearchService accepts gRPC and REST requests
- [x] JWT authentication enforced
- [x] Multi-phase search with early termination
- [x] Address collapsing removes house number spam
- [x] Ranking: confidence DESC, distance ASC
- [x] Mobile app uses SearchService
