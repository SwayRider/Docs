# REQUIREMENT_002 — Tasks Index

## Tasks

| ID | Name | Status | Description |
|----|------|--------|-------------|
| TASK_001 | iOS Project Setup | Planned | Xcode project and configuration |
| TASK_002 | KMP Integration | Planned | Import shared module |
| TASK_003 | Map UI | Planned | MapLibre SwiftUI integration |
| TASK_004 | Navigation UI | Planned | Route planning and navigation |
| TASK_005 | Auth UI | Planned | Login and registration |
| TASK_006 | Offline Support | Planned | Download and offline maps |

## Task Dependencies

```
TASK_001 (Setup) ──► TASK_002 (KMP) ──┬─► TASK_003 (Map)
                                      ├─► TASK_004 (Navigation)
                                      ├─► TASK_005 (Auth)
                                      └─► TASK_006 (Offline)
```

## Acceptance Criteria Summary

- [ ] iOS app builds
- [ ] All MVP features implemented
- [ ] Native iOS UI
- [ ] Performance acceptable
- [ ] App Store ready
