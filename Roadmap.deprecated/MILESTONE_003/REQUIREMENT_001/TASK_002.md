# TASK_002 — Mobile: RouterService API Client & Data Layer

**Status**: Planned

## Overview

Implement the Retrofit API client, DTOs, domain models, and repository for the RouterService in the Android app. The router service base URL is already configured in `build.gradle.kts` for all build variants.

## Repository

- **Repo**: swayrider
- **Subfolder**: `mobile/android/`
- **Tech**: Kotlin, Retrofit, Clean Architecture

## Background

The router service URL is already in `BuildConfig.ROUTER_SERVICE_BASE_URL`. The clean architecture pattern to follow is established by the search service:
- `data/search/` — DTOs + Retrofit interface + repository impl
- `domain/search/` — domain models + repository interface

No Retrofit interface or data/domain layer exists yet for routing.

## Technical Specification

### Domain Models (`mobile/android/app/src/main/java/.../domain/routing/`)

```kotlin
data class RouteLocation(
    val lat: Double,
    val lng: Double,
    val type: LocationType = LocationType.BREAK
)

enum class LocationType { BREAK, THROUGH, VIA }

enum class RouteType { FASTEST, SHORTEST, SCENIC }

data class RoutePreferences(
    val routeType: RouteType = RouteType.FASTEST,
    val avoidHighways: Boolean = false,
    val avoidTolls: Boolean = false,
    val avoidFerries: Boolean = false
)

data class RouteResult(
    val distanceMeters: Double,
    val durationSeconds: Double,
    val encodedPolyline: String,      // geometry for map display
    val legs: List<RouteLeg>
)

data class RouteLeg(
    val distanceMeters: Double,
    val durationSeconds: Double
)

interface RoutingRepository {
    suspend fun calculateRoute(
        origin: RouteLocation,
        destination: RouteLocation,
        preferences: RoutePreferences
    ): Result<RouteResult>
}
```

### DTOs & Retrofit (`mobile/android/app/src/main/java/.../data/routing/`)

Map to the RouterService HTTP/JSON API (gRPC-gateway). The Route endpoint is:
`POST /api/v1/router/route`

Request DTO fields to map:
- `mode` = `"RM_MOTORCYCLE"` (hardcoded for now)
- `result_mode` = `"RRM_DISPLAY"` (geometry only, no full navigation data)
- `locations[]` = array of `{lat, lng, type}`
- `route_type` = `"RT_FASTEST"` / `"RT_SHORTEST"` / `"RT_SCENIC"`
- `route_options.highway_preference` = 0.1 if avoidHighways else omit
- `route_options.tollway_preference` = 0.1 if avoidTolls else omit
- `route_options.ferry_preference` = 0.1 if avoidFerries else omit

Response DTO: parse distance, duration, and shape (encoded polyline or coordinate array) from the Valhalla trip legs.

### Dependency Injection (`SwayRiderApp.kt`)

Follow the same pattern as `searchRetrofit`:
```kotlin
val routerRetrofit = Retrofit.Builder()
    .baseUrl(BuildConfig.ROUTER_SERVICE_BASE_URL)
    .client(authHttpClientProvider.client)  // JWT auth required
    .addConverterFactory(GsonConverterFactory.create())
    .build()

val routingRepository: RoutingRepository = RoutingRepositoryImpl(
    routerRetrofit.create(RouterServiceApi::class.java)
)
```

Note: The Route endpoint is public (no JWT required on the backend), but using the auth client is fine — it just adds the header if available.

## Dependencies

- TASK_001 (route_type field must be in proto/API before mobile can send it)

## Acceptance Criteria

- [ ] `RouterServiceApi` Retrofit interface exists with `calculateRoute` method
- [ ] DTOs cover all request fields (mode, result_mode, locations, route_type, route_options)
- [ ] Response DTOs parse distance, duration, and geometry
- [ ] `RoutingRepository` interface defined in domain layer
- [ ] `RoutingRepositoryImpl` implements the interface
- [ ] Wired up in `SwayRiderApp.kt`
- [ ] Code compiles

## Testing Notes

Manually test with a hardcoded origin/destination in a debug build. Verify the API returns a valid route response from the dev server.
