# Task 03 - Comprehensive Error Case Tests + Multi-Region Verification

**Status**: closed

## Scope

- `backend/services/routerservice/internal/server/`

## Description

Add comprehensive tests to verify all acceptance criteria, including error cases and multi-region routing verification:

1. **Error case tests** — Test all error scenarios with proper gRPC status codes
2. **Multi-region verification** — Verify routes crossing 2+ regional Valhalla instances are merged correctly
3. **Motorcycle preference verification** — Verify new preference fields produce expected Valhalla costing

## Instructions

### Test File Updates

**File: `backend/services/routerservice/internal/server/route_test.go`**

Add the following test cases:

1. **Error case tests**:
   - Test Valhalla unreachable returns `UNAVAILABLE`
   - Test location outside covered regions returns `NOT_FOUND`
   - Test no route found returns `NOT_FOUND`

2. **Multi-region verification**:
   - Test that routes crossing at least two regional Valhalla instances are calculated
   - Verify returned geometry is a single merged polyline
   - Verify start/end region names are correctly populated in summary

3. **Motorcycle preference verification**:
   - Test `scenic_preference` produces appropriate Valhalla costing (use_trails, use_ferry)
   - Test `highway_avoidance` produces low use_highways
   - Test `toll_avoidance` produces low use_tolls
   - Test `unpaved_handling` enum values map correctly

### Test Approach

- Use dependency injection to mock/clamp the Valhalla client and region client
- Use table-driven tests for preference field mappings
- For multi-region tests, mock region assignment to return multiple regions

## Test Scenarios

1. **Unreachable Valhalla** — Verify gRPC `UNAVAILABLE` status
2. **Outside regions** — Verify gRPC `NOT_FOUND` status
3. **No route found** — Verify gRPC `NOT_FOUND` status
4. **Multi-region merge** — Verify single geometry for 2+ regions
5. **Region summary** — Verify start/end region names populated
6. **Scenic preference** — Verify costing produces scenic routes
7. **Highway avoidance** — Verify low use_highways
8. **Toll avoidance** — Verify low use_tolls
9. **Unpaved handling** — Verify correct exclusion/inclusion

## Acceptance Criteria

- [ ] A request with scenic preference set returns a route preferring winding roads (verified via Valhalla costing config)
- [ ] A route crossing at least two regional Valhalla instances is calculated and returned as a single merged geometry
- [ ] A request with origin or destination outside all covered regions returns a well-formed gRPC error (`NOT_FOUND`)
- [ ] A request when a Valhalla backend is unreachable returns a well-formed gRPC error (`UNAVAILABLE`)
- [ ] All existing unit tests pass
- [ ] All new unit tests for error cases pass
