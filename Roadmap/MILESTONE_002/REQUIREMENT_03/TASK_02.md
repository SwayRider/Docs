# Task 02 - SearchServiceApi: ReverseGeocode Mobile Client

**Status**: open

## Description

Expose the `ReverseGeocode` backend endpoint in the Android app by adding a Retrofit
API method, request/response data classes, and a repository method. Wire everything
into the DI container in `SwayRiderApp.kt`.

### Instructions

**Data classes**

Create the following data classes (co-locate with existing search data classes):

```kotlin
data class ReverseGeocodeRequest(
    @SerializedName("point") val point: CoordinateDto,
    @SerializedName("size") val size: Int? = null,
    @SerializedName("language") val language: String? = null,
)

data class CoordinateDto(
    @SerializedName("lat") val lat: Double,
    @SerializedName("lon") val lon: Double,
)

data class ReverseGeocodeResponse(
    @SerializedName("results") val results: List<SearchResultDto>,
)
```

Reuse the existing `SearchResultDto` for the result items.

**File: `SearchServiceApi.kt` (Retrofit interface)**

- Add a suspend function:

```kotlin
@POST("api/v1/search/reverse")
suspend fun reverseGeocode(@Body request: ReverseGeocodeRequest): ReverseGeocodeResponse
```

**Repository**

- Add a `reverseGeocode(lat: Double, lon: Double): Result<List<SearchResultDto>>`
  method to `LocationSearchRepository` (or a dedicated `ReverseGeocodeRepository`
  if the team prefers separation).
- Wrap the Retrofit call in a `runCatching` block consistent with existing repository
  methods.
- Pass `size = 1` by default (the action sheet only needs the top result).

**File: `SwayRiderApp.kt`**

- Wire the repository into the `diContainer` following the same pattern as the
  existing `locationSearchViewModel` construction.
- No new ViewModel is created in this task; the repository will be consumed by the
  ViewModel added in TASK_05.

### Test Scenarios

1. **Successful reverse geocode** — mock Retrofit response with one result; expect
   the repository to return `Result.success` containing that result.
2. **Network error** — mock Retrofit to throw an `IOException`; expect the repository
   to return `Result.failure`.
3. **Empty results list** — mock response with `results = []`; expect
   `Result.success` with an empty list (no crash).

## Acceptance Criteria

- [ ] `ReverseGeocodeRequest`, `CoordinateDto`, and `ReverseGeocodeResponse` data
      classes exist with correct JSON field names.
- [ ] `SearchServiceApi` exposes `reverseGeocode` as a `@POST` suspend function
      targeting `api/v1/search/reverse`.
- [ ] Repository method `reverseGeocode(lat, lon)` wraps the Retrofit call and
      returns `Result<List<SearchResultDto>>`.
- [ ] Repository is instantiated in `SwayRiderApp.kt` `diContainer`.
- [ ] Unit tests cover success, network error, and empty results.
- [ ] The app compiles without errors.
