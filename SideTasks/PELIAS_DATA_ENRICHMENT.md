# Pelias Data Enrichment Plan

## Goal

Improve geocoding coverage for streets, housenumbers, and POIs across all European regions.

## Current Datasources

| Source | Layers | Notes |
|---|---|---|
| OpenStreetMap | `address`, `venue`, `street` | Core importer |
| OpenAddresses | `address` | Configured per region, gaps identified below |
| GeoNames | places, localities, admin | Full country download |
| Who's On First | admin hierarchy, postalcodes | Filtered by country code |

---

## Task 1 — Fix OpenAddresses Coverage Gaps

**Priority: HIGH** — Zero pipeline changes required; config-only fixes.

### Analysis of missing files per region

#### Germany — missing Bundesländer (affects all regions)
All region configs include 12 of 16 Bundesländer. Missing everywhere:
- `de/by/statewide.csv` — **Bavaria** (most populous, ~13M people)
- `de/ni/statewide.csv` — Lower Saxony
- `de/mv/statewide.csv` — Mecklenburg-Vorpommern
- `de/be/statewide.csv` — Berlin

#### Sweden — per-municipality only (affects all regions)
Currently configured as 13 individual municipalities. Check if `se/countrywide.csv` is available in OA; if so, replace all per-municipality entries with a single countrywide file.

#### UK (`uk-iceland` region core)
No UK or Ireland address files are configured at all:
- `gb/...` — no UK addresses (check OA for available GB files)
- `ie/countrywide.csv` — Ireland
- `im/countrywide.csv` — Isle of Man

#### Netherlands (`west-europe` region)
`nl/countrywide.csv` is **missing from core** `west-europe` (BAG dataset, extremely complete).
It is present in the `uk-iceland` overlap but not in `west-europe` core where it belongs.

#### Belgium (`west-europe` region)
Missing `be/wal/bosa-region-wallonia-fr.csv` in `west-europe` core (only `de` variant present).

#### Hungary, Moldova, Romania, Ukraine (`central-europe` region)
None of these core countries have OA files configured:
- `hu/countrywide.csv`
- `ro/countrywide.csv`
- `ua/...` (check availability)

#### South-Eastern Europe
No OA files for `al`, `gr`, `me`, `mk`, `ba`, `tr`.
Check availability in OA — some may not exist yet.

### Implementation

For each gap:
1. Verify the file exists in the OpenAddresses S3 bucket: `https://results.openaddresses.io/`
2. Add the missing entry to `core.openaddresses` (or `overlap.openaddresses`) in `config/config-dev.yml`
3. Add the same entry to any other config files (`config-mac.yml`, `config-test.yml`) that mirror the region definitions

---

## Task 2 — Add Pelias Polylines Importer

**Priority: HIGH** — Improves street name search coverage significantly. Reuses existing `.osm.pbf` files.

### What it does

`pelias-polylines` imports road geometries as `street` layer records. The OSM importer only indexes streets that have address-tagged nodes nearby; polylines catches all named roads regardless.

### Changes required

#### `data-pipeline/config/config-dev.yml`

Add `polylines` repo under `pelias.repos`:
```yaml
pelias:
  repos:
    - polylines:v2.2.0:
        npm-install: true
```

#### `data-pipeline/pipeline/pelias_pipeline.py`

1. Add symlink setup for polylines tools (same pattern as other tools)
2. Call new `import_pelias_polylines` function after `import_pelias_openstreetmap`

#### `data-pipeline/pipeline/pelias_funcs.py`

Add `import_pelias_polylines(polylines_tools_path, build_path)`:
- Runs `./bin/start` with `PELIAS_CONFIG` env var
- Same pattern as the other importers

#### `data-pipeline/config/pelias-load.json` (build-time config template)

Add `polylines` block to `imports`:
```json
"polylines": {
  "datapath": "${openstreetmap_data_path}",
  "files": ["${openstreetmap_file_name}"]
}
```

Note: the polylines importer reads directly from the `.osm.pbf` file — no separate download step needed.

---

## Task 3 — Add Overture Maps Places

**Priority: MEDIUM** — Best improvement for POI (venue) coverage. Requires a pre-processing step.

### What it does

Overture Maps Foundation publishes a quarterly `places` dataset with ~50M global POIs (restaurants, shops, fuel stations, hotels, etc.) aggregated from Meta, Microsoft, and TomTom data. Coverage is strong where OSM venues are sparse.

### Approach

Overture data is published as Parquet files on S3. The pipeline needs a conversion step to produce CSV files consumable by `pelias-csv-importer`.

### Changes required

#### New download step in pipeline

For each region, download the Overture `places` parquet for the bounding box:
- Use the Overture CLI (`overturemaps` Python package) to extract by bounding box
- Convert to CSV with columns: `id`, `name`, `lat`, `lon`, `category`, `housenumber`, `street`, `postcode`, `city`, `country`

#### `data-pipeline/config/config-dev.yml`

Add `csv` importer repo:
```yaml
- csv:v1.0.0:
    npm-install: true
```

#### `data-pipeline/pipeline/pelias_funcs.py`

Add two functions:
- `download_overture_places(region, bbox, output_path)` — calls `overturemaps download` CLI
- `import_pelias_csv(csv_tools_path, build_path)` — same pattern as other importers

#### `data-pipeline/config/pelias-load.json`

Add `csv` block to `imports`:
```json
"csv": {
  "datapath": "${overture_data_path}",
  "files": ["overture-places.csv"]
}
```

#### Region bounding boxes

Each region in `config-dev.yml` needs a `bbox` field (min_lon, min_lat, max_lon, max_lat) for the Overture download query. These can be derived from the existing `srtm` extents.

---

## Task 4 — Add GTFS Transit Stops

**Priority: LOW** — Useful for routing-adjacent searches (bus stops, train stations by name/line).

### What it does

`pelias-transit` imports stops from GTFS feeds as `venue` layer records with transit metadata (routes, agency). Enables searching "Amsterdam Centraal", "Gare du Nord", etc. by station name.

### GTFS feed sources (European coverage)

| Coverage | Feed | URL |
|---|---|---|
| Belgium (SNCB/NMBS) | national rail | `https://gtfs.irail.be/nmbs/feed/` |
| Belgium (De Lijn) | Flanders bus | via GTFS Belgium open data |
| Netherlands (NS) | national rail | `https://gtfs.ovapi.nl/ns/gtfs-nl.zip` |
| Netherlands (full OV) | all public transport | `https://gtfs.ovapi.nl/nl/gtfs-nl.zip` |
| Germany (DB) | national rail | `https://data.deutschebahn.com/` |
| France (SNCF) | national rail | `https://ressources.data.sncf.com/` |
| UK (National Rail) | national rail | `https://data.atoc.org/` |
| Nordic countries | various | `https://storage.googleapis.com/marduk-production/outbound/gtfs/` (Entur/Trafiklab) |
| EU aggregate | transitfeeds.com | per-country feeds |

### Changes required

#### `data-pipeline/config/config-dev.yml`

Add `transit` repo:
```yaml
- transit:v1.0.0:
    npm-install: true
```

Add a `gtfs_feeds` section per region with feed URLs.

#### `data-pipeline/pipeline/pelias_funcs.py`

Add:
- `download_gtfs_feeds(region, feeds, output_path)` — downloads each zip
- `import_pelias_transit(transit_tools_path, build_path)` — runs `./bin/start`

#### `data-pipeline/config/pelias-load.json`

Add `transit` block to `imports`:
```json
"transit": {
  "datapath": "${gtfs_data_path}"
}
```

---

## Implementation Order

| # | Task | Effort | Impact |
|---|---|---|---|
| 1 | Fix OA coverage gaps | Low (config only) | High |
| 2 | Pelias Polylines | Medium | High |
| 3 | Overture Places | High | Medium-High |
| 4 | GTFS Transit Stops | Medium | Medium |
