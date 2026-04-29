# REQUIREMENT_008 — Search Service

## Overview

Create a new SearchService Go backend that acts as the single search endpoint for the mobile app, handling JWT authentication, RegionService fan-out, Pelias fan-out, confidence+distance ranking, and address collapsing.

## Context

- **Original Requirement**: 26_008
- **Components**: SearchService, Mobile App
- **Priority**: High
- **Status**: Done

## Background

The Android mobile app currently calls Pelias geocoding servers and RegionService directly from the data layer. This has several problems:
- Pelias endpoints are public — no authentication
- Ranking logic lives client-side and is incorrect (distance-only; should be confidence-first)
- House-number results flood the dropdown
- Adding future search features requires app releases

## Requirements

### SearchService
- gRPC with grpc-gateway REST bridge
- JWT authentication
- Multi-phase search flow with early termination
- Address collapsing (group by street+locality, keep highest confidence)
- Ranking: confidence DESC, then distance ASC

### Search Flow
1. **Setup**: Compute extended viewport, resolve regions, determine focus point
2. **Phase 1**: Core regions WITH boundary
3. **Phase 2**: Extended regions WITH boundary
4. **Phase 3**: Core + extended regions WITHOUT boundary
5. **Phase 4**: All remaining regions WITHOUT boundary

### Mobile App Changes
- Remove direct Pelias and RegionService calls
- Add SearchServiceApi Retrofit interface
- Update LocationSearchRepositoryImpl to call SearchService

## Acceptance Criteria

1. SearchService starts and accepts gRPC/REST requests
2. JWT authentication enforced
3. Multi-phase search implemented correctly
4. Address collapsing removes duplicate house numbers
5. Ranking prioritizes confidence then distance
6. Mobile app calls SearchService instead of Pelias directly
7. Search results match previous behavior (minus house number spam)

## Affected Files

### Backend
- `backend/services/searchservice/` — New service
- `backend/protos/searchservice/` — Proto definitions

### Mobile App
- `mobile/android/app/src/main/java/.../data/search/` — Refactor to use SearchService

### Infrastructure
- Docker compose for SearchService
