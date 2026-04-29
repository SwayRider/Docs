# TASK_005 — Add UK Address Data (OS AddressBase Open)

**Status**: Planned

## Overview

Import UK address data from Ordnance Survey's free open datasets. The UK has no OpenAddresses coverage (TASK_001 confirmed `sources/gb/` does not exist) and is not covered by the Overture `addresses` theme at launch. OS provides two relevant free products; this task uses both in combination.

## Repository

- **Repo**: swayrider
- **Subfolder**: `data-pipeline/`
- **Tech**: Python, Node.js (npm), YAML

## Background

The `uk-iceland` region has OSM and WoF data for Great Britain but zero address-level coverage. This means housenumber searches ("10 Downing Street", "221B Baker Street") fall back to OSM-only results, which are incomplete.

Ordnance Survey (OS) publishes two free open datasets relevant to address geocoding:

| Dataset | Content | Format | Update frequency |
|---|---|---|---|
| **OS Open UPRN** | 40M property reference numbers with coordinates and classification | CSV | Every 6 weeks |
| **OS Open Names** | Named streets, places, postcodes with coordinates | CSV | 6-monthly |

Neither dataset alone has full address text (housenumber + street + postcode in one record). The best free option that does is **AddressBase Open** (formerly AddressBase Core), available via `data.gov.uk`. It requires a free registration but provides housenumber-level address data derived from the Royal Mail PAF.

**Recommendation**: Use **AddressBase Open** as the primary source. It is free, updated regularly, and contains the structured address fields that pelias-csv-importer expects. Use **OS Open Names** as a supplemental source for streets and named places not in AddressBase.

**Coverage**: Great Britain only (England, Scotland, Wales). Northern Ireland is not covered by OS; no free open alternative exists for NI address data.

## Download Sources

| Source | URL | Registration |
|---|---|---|
| AddressBase Open | `https://www.data.gov.uk/dataset/9b1f7f9f-c6b2-4b14-8acf-89e26bfc7b46/addressbase-open` | Free account required on OS Data Hub |
| OS Open Names | `https://osdatahub.os.uk/downloads/open/OpenNames` | Free account required |
| OS Open UPRN | `https://osdatahub.os.uk/downloads/open/OpenUPRN` | Free account required |

All three require accepting the OS OpenData Licence (free, permissive for any use).

## Technical Specification

### 1. Download approach

AddressBase Open is distributed as a set of CSV files split by grid square (e.g. `AddressBase_Open_TQ.csv`). The full GB dataset is several GB. Download and merge all files in the per-region download step.

Because the download requires OS Data Hub credentials, the pipeline should read these from environment variables:
- `OS_API_KEY` — OS Data Hub API key (obtained from free account)

Alternatively, files can be downloaded manually and placed in the pipeline download directory, with the pipeline step checking for their existence before attempting download.

### 2. `data-pipeline/config/config-mac.yml`

The `uk-iceland` region already exists. No new config structure is needed — the importer will be invoked conditionally when the region is `uk-iceland` (or when GB source files are present).

Optionally add an `os_datasets` flag to the region:
```yaml
- uk-iceland:
    os_datasets:
      addressbase_open: true
      open_names: true
```

### 3. `data-pipeline/pipeline/pelias_funcs.py`

Add two functions:

#### `download_os_addressbase(download_path) -> bool`

Downloads AddressBase Open CSV files from OS Data Hub:

```python
def download_os_addressbase(download_path: str) -> bool:
    """
    Downloads AddressBase Open from OS Data Hub.
    Requires OS_API_KEY environment variable.
    Falls back to checking for manually placed files in download_path.
    """
    api_key = os.environ.get("OS_API_KEY")
    if not api_key:
        # Check if files were placed manually
        csv_files = [f for f in os.listdir(download_path)
                     if f.startswith("AddressBase_Open_") and f.endswith(".csv")]
        if csv_files:
            print(f"    Found {len(csv_files)} manually placed AddressBase files")
            return True
        print("    OS_API_KEY not set and no manual files found — skipping UK addresses")
        return False

    # Download via OS Data Hub API
    # Full download URL format:
    # https://api.os.uk/downloads/v1/products/OpenUPRN/downloads?area=GB&format=CSV&redirect
    url = ("https://api.os.uk/downloads/v1/products/AddressBaseOpen"
           "/downloads?area=GB&format=CSV&redirect")
    out_file = os.path.join(download_path, "addressbase_open.zip")
    cmd = ["curl", "-L", "-H", f"key: {api_key}", "-o", out_file, url]
    try:
        subprocess.run(cmd, check=True)
        subprocess.run(["unzip", "-o", out_file, "-d", download_path], check=True)
        os.remove(out_file)
    except subprocess.CalledProcessError as ex:
        print(f"    AddressBase download failed: {ex}")
        return False
    return True
```

#### `convert_os_addressbase_to_pelias_csv(download_path, output_csv) -> bool`

Merges all AddressBase CSV files and converts to pelias-csv-importer format.

AddressBase Open columns relevant for geocoding:

| AddressBase column | Pelias CSV column | Notes |
|---|---|---|
| `UPRN` | `id` | Unique Property Reference Number |
| `X_COORDINATE` | `lon` | BNG Easting — must convert to WGS84 |
| `Y_COORDINATE` | `lat` | BNG Northing — must convert to WGS84 |
| `BUILDING_NUMBER` | `housenumber` | |
| `THOROUGHFARE_NAME` | `street` | |
| `POST_TOWN` | `city` | |
| `POSTCODE` | `postcode` | |
| `"gb"` | `country` | hardcoded |
| `"address"` | `layer` | hardcoded |
| `"os_addressbase"` | `source` | hardcoded |

**Coordinate conversion**: AddressBase uses British National Grid (EPSG:27700). Must convert to WGS84 (EPSG:4326) using `pyproj`:
```python
from pyproj import Transformer
transformer = Transformer.from_crs("EPSG:27700", "EPSG:4326", always_xy=True)
lon, lat = transformer.transform(easting, northing)
```

Stream all grid-square CSV files sequentially; do not load all into memory at once.

### 4. `data-pipeline/pipeline/pelias_pipeline.py`

In `_create_pelias_data`, inside the per-region loop, add after Overture import:

```python
if region.name == "uk-iceland":
    print("    Downloading OS AddressBase Open")
    os_download_path = os.path.join(
            self.config.download_dir(), "os-addressbase")
    os.makedirs(os_download_path, exist_ok=True)
    os_csv_path = os.path.join(data_path, "os", region_name)
    os.makedirs(os_csv_path, exist_ok=True)

    res = download_os_addressbase(os_download_path)
    if res:
        pelias_csv = os.path.join(os_csv_path, "gb-addresses.csv")
        res = convert_os_addressbase_to_pelias_csv(
                os_download_path, pelias_csv)
        if res:
            print("    Importing OS AddressBase (CSV)")
            res = import_pelias_csv(csv_tools_path, build_path)
            if not res:
                return False
    else:
        print("    Skipping OS AddressBase (not available)")
```

### 5. `data-pipeline/config/pelias-load.json` (build-time config template)

The `csv` block added in TASK_003 already supports multiple files. When the OS file is present, add it to the list. The simplest approach is to always list it and let the importer skip missing files:

```json
"csv": {
  "datapath": "${overture_data_path}",
  "files": [
    "overture-places.csv",
    "overture-addresses.csv",
    "${os_addresses_csv}"
  ]
}
```

Alternatively, generate the file list dynamically in `create_pelias_config` based on what files actually exist in the data path.

### 6. Python dependencies

Add to pipeline environment:
- `pyproj` — coordinate transformation (BNG → WGS84)

## Dependencies

- TASK_003 (pelias-csv-importer must already be wired up)
- OS Data Hub free account (or manually downloaded AddressBase files)

## Acceptance Criteria

- [ ] `download_os_addressbase` correctly downloads or detects manually placed files
- [ ] `download_os_addressbase` is non-fatal when `OS_API_KEY` is absent and no files are present (logs warning, skips)
- [ ] `convert_os_addressbase_to_pelias_csv` converts BNG coordinates to WGS84
- [ ] `convert_os_addressbase_to_pelias_csv` streams files without loading full dataset into memory
- [ ] Records without `THOROUGHFARE_NAME` are skipped (coordinates-only entries)
- [ ] Import step only runs for `uk-iceland` region
- [ ] Elasticsearch index contains `address` records with `source: os_addressbase` and `country: gb`
- [ ] A known London address (e.g. "221B Baker Street") resolves correctly

## Testing Notes

After pipeline run for `uk-iceland`, search for a well-known London address:

```bash
curl -X GET "localhost:39200/pelias_uk-iceland-<tag>/_search" \
  -H "Content-Type: application/json" \
  -d '{"query":{"match":{"address.street":"Baker Street"}},"size":5}'
```

Also verify NI (Northern Ireland) is absent — records should only cover England, Scotland, Wales.

## Notes on Northern Ireland

No free open address data exists for Northern Ireland. Land & Property Services NI (LPS) holds the authoritative data but does not publish it openly. OSM provides partial coverage. This gap is accepted as a known limitation.
