# TASK_003 — Add Overture Maps Places & Addresses

**Status**: Done

## Overview

Integrate two Overture Maps Foundation datasets:

- **`places`** — POI/venue data aggregated from Meta, Microsoft, and TomTom (~50M global POIs). Fills gaps where OSM venue coverage is sparse.
- **`addresses`** — Address data (200M+ records) covering many EU countries that have no OpenAddresses source: Hungary, Romania, Greece, and others. This is the primary alternative for countries identified as missing in TASK_001.

Both themes share the same download mechanism (bounding box via `overturemaps` CLI) and are imported together via `pelias-csv-importer`.

## Repository

- **Repo**: swayrider
- **Subfolder**: `data-pipeline/`
- **Tech**: Python, Node.js (npm), YAML

## Background

Overture data is distributed as Parquet files on AWS S3, partitioned by theme (`addresses`, `places`, `buildings`, etc.). The `overturemaps` Python CLI downloads a bbox-clipped GeoJSON for any theme. A conversion step produces the CSV format expected by `pelias-csv-importer`.

The `addresses` theme launched in July 2024 and covers EU countries including those missing from OpenAddresses. The `places` theme has been available since 2023 with strong EU commercial POI coverage.

## Technical Specification

### 1. Region bounding boxes

Each region in the config needs a geographic bounding box for the Overture download. Add a `bbox` field to each region entry in `config-dev.yml`:

```yaml
regions:
  - west-europe:
      bbox: [-5.5, 42.0, 17.0, 55.5]   # [min_lon, min_lat, max_lon, max_lat]
      core:
        ...
```

Derive bbox values from the existing `srtm` extents already defined per region.

### 2. `data-pipeline/config/config-mac.yml`

Add `csv` importer to `pelias.repos`:

```yaml
- csv:latest:
    npm-install: true
```

Add `overturemaps` Python dependency (install via pip in the pipeline environment or add to `requirements.txt` if one exists).

### 3. `data-pipeline/pipeline/config/region.py`

Add `bbox()` method to the `Region` class that reads the new `bbox` config field and returns `(min_lon, min_lat, max_lon, max_lat)`.

### 4. `data-pipeline/pipeline/pelias_funcs.py`

Add three functions:

#### `_download_overture_theme(bbox, theme, geojson_file) -> bool`

Shared download helper for any Overture theme:

```python
def _download_overture_theme(
        bbox: tuple[float, float, float, float],
        theme: str,
        geojson_file: str) -> bool:
    min_lon, min_lat, max_lon, max_lat = bbox
    cmd = [
        "overturemaps", "download",
        "--bbox", f"{min_lon},{min_lat},{max_lon},{max_lat}",
        "-f", "geojson",
        "--type", theme,
        "-o", geojson_file
    ]
    try:
        subprocess.run(cmd, check=True)
    except subprocess.CalledProcessError as ex:
        print(ex)
        return False
    return True
```

#### `download_overture_data(bbox, output_path) -> bool`

Downloads both `place` and `address` themes and converts each to a pelias-csv-importer CSV:

```python
def download_overture_data(
        bbox: tuple[float, float, float, float],
        output_path: str) -> bool:
    os.makedirs(output_path, exist_ok=True)

    # Places (venues/POIs)
    places_geojson = os.path.join(output_path, "overture-places.geojson")
    places_csv = os.path.join(output_path, "overture-places.csv")
    if not _download_overture_theme(bbox, "place", places_geojson):
        return False
    _convert_overture_places_to_csv(places_geojson, places_csv)
    os.remove(places_geojson)

    # Addresses
    addresses_geojson = os.path.join(output_path, "overture-addresses.geojson")
    addresses_csv = os.path.join(output_path, "overture-addresses.csv")
    if not _download_overture_theme(bbox, "address", addresses_geojson):
        return False
    _convert_overture_addresses_to_csv(addresses_geojson, addresses_csv)
    os.remove(addresses_geojson)

    return True
```

#### CSV conversion functions

Both stream line-by-line (newline-delimited GeoJSON) to avoid loading large files into memory.

**`_convert_overture_places_to_csv(geojson_file, csv_file)`** — `venue` layer:

| CSV column | Overture source |
|---|---|
| `id` | `properties.id` |
| `name` | `properties.names.primary` |
| `lat` | `geometry.coordinates[1]` |
| `lon` | `geometry.coordinates[0]` |
| `layer` | `"venue"` (hardcoded) |
| `source` | `"overture"` (hardcoded) |
| `category` | `properties.categories.primary` |
| `housenumber` | `properties.addresses[0].freeform` |
| `street` | `properties.addresses[0].street` |
| `postcode` | `properties.addresses[0].postcode` |
| `city` | `properties.addresses[0].locality` |
| `country` | `properties.addresses[0].country` |

**`_convert_overture_addresses_to_csv(geojson_file, csv_file)`** — `address` layer:

| CSV column | Overture source |
|---|---|
| `id` | `properties.id` |
| `lat` | `geometry.coordinates[1]` |
| `lon` | `geometry.coordinates[0]` |
| `layer` | `"address"` (hardcoded) |
| `source` | `"overture"` (hardcoded) |
| `housenumber` | `properties.number` |
| `street` | `properties.street` |
| `postcode` | `properties.postcode` |
| `city` | `properties.city` |
| `country` | `properties.country` |

Skip records with no `properties.street` (coordinates-only entries are not useful for address search).

#### `import_pelias_csv(csv_tools_path, build_path) -> bool`

Same pattern as other importers:

```python
def import_pelias_csv(
        csv_tools_path: str,
        build_path: str) -> bool:
    config_file_path = os.path.join(build_path, "pelias.json")

    cmd0 = ["npm", "install", "pelias-config"]
    cmd1 = ["./bin/start"]

    env = os.environ.copy()
    env["PELIAS_CONFIG"] = config_file_path

    try:
        subprocess.run(cmd0, cwd=csv_tools_path, check=True)
        subprocess.run(cmd1, cwd=csv_tools_path, env=env, check=True)
    except subprocess.CalledProcessError as ex:
        print(ex)
        return False

    return True
```

### 5. `data-pipeline/pipeline/pelias_pipeline.py`

In `_create_pelias_data`, add before the per-region loop:

```python
csv_tools_path = self._symlink_path(
        self.tools["pelias"]["csv"], "pelias-tools-csv")
```

Inside the per-region loop, after OSM/polylines imports:

```python
print("    Downloading Overture data (places + addresses)")
overture_data_path = os.path.join(data_path, "overture", region_name)
os.makedirs(overture_data_path, exist_ok=True)
res = download_overture_data(region.bbox(), overture_data_path)
if not res:
    print("    Warning: Overture download failed, continuing without it")
    # Non-fatal: OSM still provides baseline coverage

print("    Importing Overture data (CSV)")
res = import_pelias_csv(csv_tools_path, build_path)
if not res:
    return False
```

### 6. `data-pipeline/config/pelias-load.json` (build-time config template)

Add a `csv` block to `imports` referencing both output files:

```json
"csv": {
  "datapath": "${overture_data_path}",
  "files": [
    "overture-places.csv",
    "overture-addresses.csv"
  ]
}
```

Add `overture_data_path` to the template variables in `create_pelias_config` in `pelias_funcs.py`.

## Dependencies

- `overturemaps` Python package must be installed in the pipeline environment
- TASK_001 (recommended first — OA config fixes avoid duplicate address records where both OA and Overture have coverage)

## Acceptance Criteria

- [ ] `bbox` field added to all regions in `config-mac.yml`
- [ ] `csv:latest` added to `pelias.repos` in `config-mac.yml`
- [ ] `Region.bbox()` method implemented
- [ ] `download_overture_data` downloads both `place` and `address` themes for a test region
- [ ] Both CSV converters stream without loading full file into memory
- [ ] Address records with no `street` are skipped
- [ ] `import_pelias_csv` runs the pelias-csv-importer successfully
- [ ] `pelias-load.json` template references both CSV files
- [ ] Elasticsearch index contains `venue` records with `source: overture` after pipeline run
- [ ] Elasticsearch index contains `address` records with `source: overture` for countries missing from OA (e.g. Hungary)
- [ ] Overture download failure is non-fatal (warning logged, pipeline continues)

## Testing Notes

Run on a small region covering a country missing from OA (e.g. `central-europe` to get Hungary). After import:

```bash
# Check Overture venues
curl -X GET "localhost:39200/pelias_<region>-<tag>/_search" \
  -H "Content-Type: application/json" \
  -d '{"query":{"bool":{"must":[{"term":{"source":"overture"}},{"term":{"layer":"venue"}}]}},"size":3}'

# Check Overture addresses (should include Hungary)
curl -X GET "localhost:39200/pelias_<region>-<tag>/_search" \
  -H "Content-Type: application/json" \
  -d '{"query":{"bool":{"must":[{"term":{"source":"overture"}},{"term":{"layer":"address"}}]}},"size":3}'
```

Verify address records include `housenumber`, `street`, and `country` fields.
