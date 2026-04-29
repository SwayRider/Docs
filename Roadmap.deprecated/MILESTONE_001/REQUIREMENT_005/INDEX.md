# REQUIREMENT_005 — Tasks Index

## Tasks

| ID | Name | Status | Description |
|----|------|--------|-------------|
| TASK_001 | Highway Reference Extraction | Done | Extract A and E highway numbers from OSM |
| TASK_002 | Highway Label Styling | Done | Create blue/green box styles for highway refs |
| TASK_003 | Street Name Data | Done | Ensure street names in tile data from Z15+ |
| TASK_004 | Street Label Styling | Done | Add street name label layers to styles |

## Task Dependencies

```
TASK_001 (Highway Data) ──► TASK_002 (Highway Style)
TASK_003 (Street Data) ──► TASK_004 (Street Style)
```

## Acceptance Criteria Summary

- [x] Highway A-numbers in blue boxes from Z7
- [x] Highway E-numbers in green boxes from Z7
- [x] Street names for major roads from Z15
- [x] Street names for minor roads from Z16
- [x] Labels follow road orientation
