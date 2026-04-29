# TASK_004 — Add GTFS Transit Stops

**Status**: Done

## Overview

Add `pelias-transit` as an importer to index public transport stops (bus, tram, metro, rail) as `venue` layer records. Enables searching for stops by name (e.g. "Amsterdam Centraal", "Gare du Nord") and provides transit stop awareness for routing-adjacent features.

## Repository

- **Repo**: swayrider
- **Subfolder**: `data-pipeline/`
- **Tech**: Python, Node.js (npm), YAML

## Background

OSM already contains many transit stops, but GTFS feeds are the authoritative source: they include official stop names, IDs, route associations, and agency data that OSM data often lacks or has inconsistently. GTFS data is updated frequently (some feeds weekly), making it a good complement to the quarterly OSM/Overture cycle.

## GTFS Feed Sources

The following open feeds cover the European regions in scope. Feed URLs should be added to the region config — they are all freely accessible without authentication.

| Region | Agency | Feed URL |
|---|---|---|
| Belgium | iRail (SNCB/NMBS) | `https://gtfs.irail.be/nmbs/feed/gtfs.zip` |
| Belgium | De Lijn (bus/tram Flanders) | `https://opendata.delijn.be/gtfs/google_transit.zip` |
| Belgium | STIB/MIVB (Brussels) | `https://stibmivb.opendatasoft.com/api/explore/v2.1/catalog/datasets/gtfs-files-production/files/` |
| Netherlands | NS (national rail) | `https://gtfs.ovapi.nl/ns/gtfs-nl.zip` |
| Netherlands | all OV | `https://gtfs.ovapi.nl/nl/gtfs-nl.zip` |
| France | SNCF (national) | via `https://data.sncf.com/` open data portal |
| Germany | DB (national) | via `https://data.deutschebahn.com/` open data portal |
| UK | ATOC (national rail) | `https://data.atoc.org/rail-industry-data` (requires free registration) |
| Norway | Entur (all PT) | `https://storage.googleapis.com/marduk-production/outbound/gtfs/rb_norway-aggregated-gtfs.zip` |
| Sweden | Trafiklab | per-operator, `https://www.trafiklab.se/` (API key required) |
| Denmark | Rejseplanen | `https://www.rejseplanen.dk/` open data |
| Finland | Fintraffic | `https://www.fintraffic.fi/en/opendata` |
| Switzerland | Open Data Platform Mobility | `https://opentransportdata.swiss/en/dataset/timetable-2024-gtfs2020` |
| Austria | ÖBB | via `https://data.oebb.at/` |
| Italy | various | check `https://gtfs.guide/` for regional feeds |
| Spain | various | RENFE and regional operators |
| Poland | various | per-city GTFS feeds |

**Note**: Some feeds (Sweden Trafiklab, UK ATOC) require free registration or API keys. Document these in a `data-pipeline/config/gtfs-sources.md` file.

## Technical Specification

### 1. `data-pipeline/config/config-dev.yml`

Add `transit` to `pelias.repos`:

```yaml
- transit:v1.0.0:
    npm-install: true
```

Add a `gtfs_feeds` list to each region entry:

```yaml
regions:
  - west-europe:
      gtfs_feeds:
        - https://gtfs.irail.be/nmbs/feed/gtfs.zip
        - https://opendata.delijn.be/gtfs/google_transit.zip
        - https://gtfs.ovapi.nl/nl/gtfs-nl.zip
        - <france-sncf-url>
        - <de-db-url>
      core:
        ...
```

### 2. `data-pipeline/pipeline/config/region.py`

Add `gtfs_feeds()` method to `Region` that returns the list from config (empty list if not configured):

```python
def gtfs_feeds(self) -> list[str]:
    return self._gtfs_feeds

# in _parse():
self._gtfs_feeds = config.get("gtfs_feeds", [])
```

### 3. `data-pipeline/pipeline/pelias_funcs.py`

Add two functions:

#### `download_gtfs_feeds(feeds, output_path) -> bool`

Downloads each GTFS zip into `output_path`:

```python
def download_gtfs_feeds(
        feeds: list[str],
        output_path: str) -> bool:
    os.makedirs(output_path, exist_ok=True)
    for i, url in enumerate(feeds):
        out_file = os.path.join(output_path, f"feed_{i:03d}.zip")
        if os.path.exists(out_file):
            print(f"    Skipping {url} (already downloaded)")
            continue
        try:
            subprocess.run(
                ["curl", "-L", "-o", out_file, url],
                check=True)
        except subprocess.CalledProcessError as ex:
            print(f"    Failed to download {url}: {ex}")
            return False
    return True
```

#### `import_pelias_transit(transit_tools_path, build_path) -> bool`

Same pattern as other importers:

```python
def import_pelias_transit(
        transit_tools_path: str,
        build_path: str) -> bool:
    config_file_path = os.path.join(build_path, "pelias.json")

    cmd0 = ["npm", "install", "pelias-config"]
    cmd1 = ["./bin/start"]

    env = os.environ.copy()
    env["PELIAS_CONFIG"] = config_file_path

    try:
        subprocess.run(cmd0, cwd=transit_tools_path, check=True)
        subprocess.run(cmd1, cwd=transit_tools_path, env=env, check=True)
    except subprocess.CalledProcessError as ex:
        print(ex)
        return False

    return True
```

### 4. `data-pipeline/pipeline/pelias_pipeline.py`

In `_create_pelias_data`:

```python
transit_tools_path = self._symlink_path(
        self.tools["pelias"]["transit"], "pelias-tools-transit")
```

Inside the per-region loop, skip if no feeds configured:

```python
feeds = region.gtfs_feeds()
if feeds:
    print("    Downloading GTFS feeds")
    gtfs_data_path = os.path.join(data_path, "gtfs", region_name)
    os.makedirs(gtfs_data_path, exist_ok=True)
    res = download_gtfs_feeds(feeds, gtfs_data_path)
    if not res:
        return False

    print("    Importing GTFS transit stops")
    res = import_pelias_transit(transit_tools_path, build_path)
    if not res:
        return False
else:
    print("    Skipping GTFS (no feeds configured)")
```

### 5. `data-pipeline/config/pelias-load.json` (build-time config template)

Add a `transit` block to `imports`:

```json
"transit": {
  "datapath": "${gtfs_data_path}"
}
```

Add `gtfs_data_path` to the template variables in `create_pelias_config` in `pelias_funcs.py`. When no feeds are configured, set `gtfs_data_path` to an empty temp dir (the importer handles an empty directory gracefully).

### 6. New file: `data-pipeline/config/gtfs-sources.md`

Document all feed sources, update frequencies, registration requirements, and feed URLs. This serves as the reference for future updates.

## Dependencies

- TASK_001, TASK_002 (recommended — complete the foundational improvements first)

## Acceptance Criteria

- [ ] `transit:v1.0.0` added to `pelias.repos` in all config files
- [ ] `gtfs_feeds` field added to at least the `west-europe` and `uk-iceland` regions
- [ ] `Region.gtfs_feeds()` method implemented
- [ ] `download_gtfs_feeds` downloads all configured feeds for a test region
- [ ] `import_pelias_transit` runs the pelias-transit importer successfully
- [ ] `pelias-load.json` template includes `transit` import block
- [ ] Regions without `gtfs_feeds` configured are skipped without error
- [ ] Elasticsearch index contains `venue` records with transit stop names after pipeline run
- [ ] `data-pipeline/config/gtfs-sources.md` documents all feed URLs and access requirements

## Testing Notes

After pipeline run, query for transit venues:

```bash
curl -X GET "localhost:39200/pelias_<region>-<tag>/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {"term": {"source": "transit"}}, "size": 5}'
```

Verify records include stop names, coordinates, and route associations.
