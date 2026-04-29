# TASK_002 — Add Pelias Polylines Importer

**Status**: Done

## Overview

Add `pelias-polylines` as an additional importer step in the Pelias pipeline. This reads road geometries directly from the existing `.osm.pbf` files and indexes all named streets as `street` layer records — including roads that the OSM importer misses because they lack nearby address-tagged nodes.

## Repository

- **Repo**: swayrider
- **Subfolder**: `data-pipeline/`
- **Tech**: Python, Node.js (npm), YAML

## Background

The current OSM importer (`pelias-openstreetmap`) indexes streets only when it encounters address data (`addr:*` tags) on nearby nodes or ways. Roads without any address context are silently skipped. The `pelias-polylines` importer takes a different approach: it reads every named way geometry from the PBF and creates a `street` record for each, dramatically improving street name search recall.

The `.osm.pbf` files are already produced by `build-osm` and available at `result/osm/<region>.osm.pbf`. No extra download is needed.

## Technical Specification

### 1. `data-pipeline/config/config-dev.yml` (and `config-mac.yml`)

Add `polylines` to the `pelias.repos` list:

```yaml
pelias:
  repos:
    - polylines:v2.2.0:
        npm-install: true
```

### 2. `data-pipeline/pipeline/pelias_funcs.py`

Add a new function following the exact same pattern as `import_pelias_openstreetmap`:

```python
def import_pelias_polylines(
        polylines_tools_path: str,
        build_path: str) -> bool:
    config_file_path = os.path.join(build_path, "pelias.json")

    cmd0 = ["npm", "install", "pelias-config"]
    cmd1 = ["./bin/start"]

    env = os.environ.copy()
    env["PELIAS_CONFIG"] = config_file_path

    try:
        subprocess.run(cmd0, cwd=polylines_tools_path, check=True)
        subprocess.run(cmd1, cwd=polylines_tools_path, env=env, check=True)
    except subprocess.CalledProcessError as ex:
        print(ex)
        return False

    return True
```

### 3. `data-pipeline/pipeline/pelias_pipeline.py`

In `_create_pelias_data`:

a) Add symlink setup alongside the other tools:
```python
polylines_tools_path = self._symlink_path(
        self.tools["pelias"]["polylines"], "pelias-tools-polylines")
```

b) Call the importer after `import_pelias_openstreetmap`:
```python
print("    Importing polylines data")
res = import_pelias_polylines(
        polylines_tools_path, build_path)
if not res:
    return False
```

c) Add the import at the top of the file:
```python
from .pelias_funcs import import_pelias_polylines
```

### 4. `data-pipeline/config/pelias-load.json` (build-time config template)

Add a `polylines` block inside `imports`. The polylines importer reads directly from the OSM PBF, so it reuses the same `openstreetmap_data_path` and `openstreetmap_file_name` template variables already defined:

```json
"polylines": {
  "datapath": "${openstreetmap_data_path}",
  "files": ["${openstreetmap_file_name}"]
}
```

Note: `pelias-prod.json` does not need a `polylines` block — the importer is only used at build time.

## Dependencies

- `build-osm` must have run first (the `.osm.pbf` prerequisite check in `pelias_pipeline.py` already enforces this)

## Acceptance Criteria

- [ ] `polylines:v2.2.0` (or latest compatible) added to `pelias.repos` in all config files
- [ ] `import_pelias_polylines` function exists in `pelias_funcs.py`
- [ ] Polylines import step called after OSM import in `pelias_pipeline.py`
- [ ] `pelias-load.json` template includes `polylines` import block pointing to the PBF file
- [ ] Pipeline runs to completion without error on a test region
- [ ] Elasticsearch index contains `street` records with `source: openstreetmap` from the polylines import

## Testing Notes

After the pipeline runs, query Elasticsearch for streets that were previously missing:
```bash
curl -X GET "localhost:39200/pelias_<region>-<tag>/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {"bool": {"must": [{"match": {"layer": "street"}}]}},"size": 5}'
```

Compare street record count before and after to confirm improvement.
