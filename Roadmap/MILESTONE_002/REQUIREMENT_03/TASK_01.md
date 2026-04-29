# Task 01 - ReverseGeocode RPC: Proto & SearchService Backend

**Status**: closed

## Description

Add a `ReverseGeocode` RPC to SearchService that resolves a geographic coordinate
to a human-readable address using the Pelias `/v1/reverse` endpoint. Region routing
reuses the existing `regionClient.SearchBox` mechanism with a minimal bounding box
around the requested coordinate.

### Instructions

**File: `backend/protos/searchservice/v1/search.proto`**

- Add the following messages:

```protobuf
message ReverseGeocodeRequest {
  common_types.Coordinate point    = 1;
  optional int32          size     = 2;
  optional string         language = 3;
}

message ReverseGeocodeResponse {
  repeated Result results = 1;
}
```

- Add the RPC to the `SearchService` service definition:

```protobuf
rpc ReverseGeocode(ReverseGeocodeRequest) returns (ReverseGeocodeResponse) {
  option (google.api.http) = {
    post: "/api/v1/search/reverse"
    body: "*"
  };
}
```

- Run `make proto` from the repository root to regenerate Go bindings.

**File: `backend/services/searchservice/internal/pelias/client.go`**

- Add a `Reverse` method to the existing `Client` struct:

```go
func (c *Client) Reverse(ctx context.Context, lat, lon float64, size int, language string) (*Response, error)
```

- The method must call the Pelias `/v1/reverse` endpoint with query parameters
  `point.lat`, `point.lon`, `size`, and `lang`.
- Follow the same HTTP call, error-handling, and response-parsing conventions as
  the existing `Search` method on the same client.

**File: `backend/services/searchservice/internal/server/server.go`**

- Implement the `ReverseGeocode` gRPC handler.
- Use `regionClient.SearchBox` with a bounding box of ±0.001 degrees around the
  requested coordinate to identify the correct Pelias region URL.
- Construct a `pelias.Client` for that region URL and call `Reverse`.
- Map the Pelias response to `[]searchv1.Result` using the existing result-mapping
  helper.
- Return a `ReverseGeocodeResponse` containing the mapped results.
- No `security` registration is required: the zero-value `EndpointProfile` (secure,
  JWT required) applies automatically.

### Test Scenarios

1. **Valid coordinate in a known region** — mock Pelias to return one result; expect
   a `ReverseGeocodeResponse` with one mapped `Result`.
2. **No region found for coordinate** — `regionClient.SearchBox` returns empty list;
   expect an appropriate gRPC error (`NotFound` or `InvalidArgument`).
3. **Pelias returns empty results** — mock Pelias `/v1/reverse` with an empty feature
   collection; expect a `ReverseGeocodeResponse` with zero results (no error).
4. **Pelias HTTP error** — mock Pelias to return a 500; expect a gRPC `Internal` error.
5. **`size` and `language` optional fields** — when omitted, the Pelias request must
   not include those query parameters.

## Acceptance Criteria

- [x] `search.proto` contains `ReverseGeocodeRequest`, `ReverseGeocodeResponse`, and
      the `ReverseGeocode` RPC with HTTP gateway annotation.
- [x] `make proto` runs without errors and regenerates Go bindings in
      `backend/protos/searchservice/v1/`.
- [x] `pelias.Client` exposes a `Reverse(ctx, lat, lon, size, language)` method that
      calls `/v1/reverse`.
- [x] `ReverseGeocode` handler routes to the correct Pelias region using
      `regionClient.SearchBox`.
- [x] Unit tests for `pelias.Client.Reverse` cover success, empty results, and HTTP
      error cases.
- [x] All existing tests in `backend/services/searchservice/...` continue to pass.
- [x] `go build ./...` in `backend/` compiles without errors.
