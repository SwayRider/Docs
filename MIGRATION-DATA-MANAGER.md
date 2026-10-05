# Migration: data-pipeline → data-manager

Status: **planning** (written 2026-10-05). Owner: platform. Test-bed: `infra/dev-mini` (single-server docker-compose).

`data-manager/` replaces the Python scripts in `data-pipeline/`. This document is the single place that lists every step needed to move SwayRider over: what changes in each service, in infra, how data reaches a target server, and the order to do it in.

Related documents:

- [`data-manager/CLAUDE.md`](../data-manager/CLAUDE.md), [`DESIGN.md`](../data-manager/DESIGN.md), [`SERVICES.md`](../data-manager/SERVICES.md): data-manager design and what it expects from each downstream service.
- [`data-manager/TILESSERVICE-PMTILES.md`](../data-manager/TILESSERVICE-PMTILES.md): detailed tilesservice handover (PR 0–5).
- [`TECHNOLOGY_DECISIONS.md`](TECHNOLOGY_DECISIONS.md): stack rationale (updated for this migration).

---

## 1. Overview

### 1.1 What changes

| Concern | data-pipeline (legacy, deprecated) | data-manager (new) |
|---|---|---|
| Run model | Five CLI scripts, YAML config, one manifest per pipeline | Flask UI + RQ worker, SQLite state, stages with `produces`/`consumes` assets, approve/reject per run |
| Config | `config/config-*.yml` | Map UI (countries → regions → overlap/borders), exported to legacy YAML shape on demand |
| OSM source | Geofabrik downloads per region | Planet download + one-pass `.poly` extraction per country (Geofabrik still selectable) |
| Tiles | Hand-built MBTiles (tippecanoe), L0/L1/L2 files | Protomaps **planet PMTiles v3** (z0–15, gzip MVT), downloaded, plus styles/glyphs/sprites |
| Pelias | ES snapshot + data tarball | `pelias-index-snapshot`, `pelias-config`, `pelias-wof` (patched WOF), **`pelias-interpolation`** (`street.db`, `address.db`), transit (GTFS), Overture/OpenAddresses CSV |
| Valhalla | `valhalla.tar.bz2` | `valhalla-tiles` (`tiles.tar`), `valhalla-admin`, `valhalla-timezones`, `valhalla-polylines` per region |
| Border | `border.tar.bz2` | `region-outline` + `border-crossings` assets (same CSV columns and manifest) |
| Distribution | Tagged tarballs in a geodata store + `infra/*/scripts/deploy-*.sh` extract them | **Release directories copied to the target** and activated with a `current` symlink (see §3) |
| Machine API | n/a | None yet (UI only). Deploy/publish is Phase 3 and **not built** |

### 1.2 Artifact flow

```
 BUILD HOST (data-manager; may be a different machine)          TARGET HOST (dev-mini, docker compose)
 ┌───────────────────────────────────────────────┐              ┌──────────────────────────────────────────┐
 │ DATA_ROOT/                                    │              │ TILES_ROOT/     current -> releases/<id>  │──► tilesservice (ro)
 │  downloads/  library/assets/  work/  tools/   │   rsync/ssh  │ VALHALLA_ROOT/  current -> releases/<id>  │──► valhalla-<region> (ro)
 │  releases/<id>/{tiles,valhalla,pelias,geodata}│ ───────────► │ PELIAS_ROOT/    current -> releases/<id>  │──► pelias-pip/api/interpolation (ro)
 │  state/db.sqlite3   (Flask + RQ worker)       │  checksum    │ GEODATA_ROOT/   current -> releases/<id>  │──► regionservice (ro)
 └───────────────────────────────────────────────┘  verified    │ ES_SNAPSHOTS_PATH/  (snapshot restore)    │──► elasticsearch
                                                                └──────────────────────────────────────────┘
```

### 1.3 Per-artifact impact matrix

| Artifact | Consumer | Change required | Where |
|---|---|---|---|
| Planet PMTiles + styles/glyphs/sprites | tilesservice | **Rewrite tile backend** (MBTiles → PMTiles), release reload, new routes | §5.1 |
| Tiles traffic | swayrider-api | No routing change; add `tiles` rate-limit class, transport tuning, doc fixes | §5.2 |
| Border outlines + crossings | regionservice | None (keep `manifest.yml` format and file layout) | §5.3 |
| Valhalla tiles per region | Valhalla containers → routerservice | Rename `tiles.tar` → `valhalla_tiles.tar` at deploy; mounts move to `current` | §5.4 |
| Pelias ES snapshot / WOF / config | Elasticsearch, pelias-pip, pelias-api → searchservice | Snapshot restore + alias switch; PIP reads patched WOF; API config updated | §5.5 |
| Interpolation DBs | **new** `pelias-interpolation` service | New compose service (port 4300) + `interpolation.client` in API `pelias.json` | §5.5 |
| Everything | infra/dev-mini | New env vars/roots, mounts, new service, README, data-manager compose | §3, §6 |

---

## 2. Current status and gaps

Built in data-manager (phases 0–6): configuration UI, downloads, OSM, border, Valhalla, styles (single light/dark pair), Pelias (+ WOF patch, Overture/GTFS, interpolation), tiles download.

**Not built / must be done before cutover:**

1. **Publish + deploy (Phase 3):** no assembly, `publish_record`, `deployment_record`, `releases/<id>` writer, active-release symlink or rollback. `releases/` and `deploy-state/` in `DATA_ROOT` are reserved but empty. Decision (2026-10): deployment is **copy-based** (§3). Until built, the copy runs from the scripts in §3.4.
2. **Styles stage** must grow: multiple named styles, a style version, `manifest.json`, glyph and sprite assets.
3. **`download.tiles_keep`** defaults to 1; rollback needs ≥ 2 (≈ 140 GB each).
4. **Pelias interpolation service** and a deploy-side PIP service do not exist in infra.
5. **No Dockerfile/compose** for data-manager itself; CI is DCO-only (no tests/build).
6. **Unverified:** Valhalla stage against a real build; border stage vs a legacy run on real data; Protomaps terms for repeated automated downloads and public serving; MapLibre `maxzoom: 15` over-zoom behaviour; WOF patch only spiked on Belgium and Germany.
7. **Known hazards:** csv-importer/transit importers exit 0 after rejecting config (judge by documents added); SQLite concurrency with several heavy jobs unconfirmed; ES needs `vm.max_map_count >= 262144`.

---

## 3. Deployment strategy (dev-mini as single-server test-bed)

### 3.1 Principles

- **data-manager builds and publishes releases; it does not need to run on the target.** Deploying is always a *copy* (rsync over ssh, resumable, checksum-verified). A target host never needs Python, osmium, tippecanoe or GDAL.
- **One root per artifact class**, each configured by an env var so it can sit on its own drive (tiles ≈ 140 GB × 2 on a large SSD; ES data on the fastest; Valhalla/Pelias/geodata elsewhere).
- **Immutable releases + atomic switch:** `<ROOT>/releases/<release-id>/…` plus a relative symlink `<ROOT>/current -> releases/<release-id>`. Services mount `<ROOT>` read-only (or `current`) and never see a half-copied release.
- **Keep the last 2 releases** per class for rollback; older ones are pruned by the deploy script.
- **Activation is per class** and explicit (SIGHUP/poll, container restart, snapshot restore). It is a separate step from copy so a copy can be done ahead of time.

### 3.2 Per-class roots (env vars, `infra/dev-mini`)

| Variable | Example mount | Content | Size guide | Notes |
|---|---|---|---|---|
| `TILES_ROOT` | `/mnt/ssd-a/swayrider/tiles` | PMTiles releases (+ legacy `base/` during transition) | ~140 GB × 2 | Replaces `TILES_DATA_PATH`; `TILES_CACHE_PATH` removed |
| `VALHALLA_ROOT` | `/mnt/ssd-b/swayrider/valhalla` | per-region tiles, admin/tz sqlite | tens of GB | Replaces `VALHALLA_DATA_PATH` |
| `PELIAS_ROOT` | `/mnt/ssd-b/swayrider/pelias` | WOF sqlite, interpolation DBs, `pelias.json`, placeholder | tens of GB | Replaces `PELIAS_DATA_PATH` |
| `GEODATA_ROOT` | `/mnt/ssd-b/swayrider/geodata` | `manifest.yml`, contours, border-crossings | < 1 GB | Replaces `GEODATA_PATH` |
| `ES_SNAPSHOTS_PATH` | `/mnt/ssd-c/swayrider/es-snapshots` | snapshot repo (existing var) | per-region snapshots | Restore source |
| `ES_DATA_PATH` | `/mnt/ssd-c/swayrider/es-data` | live ES data (existing var) | 100s of GB for 3 regions | Fastest drive |
| `DATA_ROOT` (build host only) | `/data/swayrider-dm` | data-manager working set | large | Not on the target |

### 3.3 Folder structure

Target host (example for dev-mini: benelux, france, germany):

```
$TILES_ROOT/
  current -> releases/20261012T0900Z
  releases/
    20261012T0900Z/
      tiles.pmtiles
      manifest.json
      styles/<style-id>/<version>/{light,dark}.json
      glyphs/<fontstack>/<range>.pbf
      sprites/<name>[@2x].{json,png}
    20261005T1200Z/            # previous (rollback)
  base/                        # legacy MBTiles L0.mbtiles, L1/, L2/ (transition only, removed in Phase G)

$VALHALLA_ROOT/
  current -> releases/<id>
  releases/<id>/<region>/{valhalla_tiles.tar, admin.sqlite, tz_world.sqlite}      # region ∈ benelux|france|germany

$PELIAS_ROOT/
  current -> releases/<id>
  releases/<id>/
    placeholder/data/
    <region>/wof/                       # patched WOF sqlite/ dir, read by pelias-pip
    <region>/interpolation/{street.db,address.db}
    <region>/pelias.json                # production config (ES host `elasticsearch`, WOF path /data/whosonfirst)

$GEODATA_ROOT/
  current -> releases/<id>
  releases/<id>/{manifest.yml, contours/<region>-{core|extended}.geojson, border-crossings/<a>--<b>.csv}

$ES_SNAPSHOTS_PATH/<release-id>/<region>/…      # copied, then restored into ES; alias switched
```

Build host (`DATA_ROOT`, from `datamanager/config.py`): `state/`, `downloads/`, `library/`, `work/`, `tools/`, `releases/<id>/{tiles,valhalla,pelias,geodata}` (publish output), `deploy-state/`.

Region names must be identical across `GEODATA_ROOT` manifest, `VALHALLA_REGION_*`, `PELIAS_REGIONS` and the directory names above.

### 3.4 Copy + activate procedure

Data-manager models this as **package** (tagged archive) + **deploy configuration** (pluggable driver; first `compose-single-machine`) = **deploy**; spec in [`data-manager/RELEASE-CONTRACT.md`](../data-manager/RELEASE-CONTRACT.md) (implemented on the data-manager server, not on the dev machine). Until then, and as a manual fallback, implement as scripts in `infra/dev-mini/scripts/` (replacing the legacy `deploy-*.sh`, which are deprecated):

```
deploy.sh copy     <class> <release-id> --from <build-host:/path/releases/<id>/<class>> --to <ROOT>
deploy.sh activate <class> <release-id>
deploy.sh rollback <class>
deploy.sh list
```

`copy`:

1. `rsync -a --partial --inplace-safe --checksum` into `<ROOT>/releases/<id>.partial/` (ssh, resumable).
2. Verify every file against the release `manifest.json` hash list; abort on mismatch.
3. `mv <id>.partial <id>` (atomic on the same filesystem).

`activate` (atomic switch, then service action):

| Class | Switch | Service action |
|---|---|---|
| tiles | `ln -sfn releases/<id> current.tmp && mv -T current.tmp current` | tilesservice reloads on SIGHUP or by polling the symlink; **no restart**, in-flight requests finish on the old file. `/ready` confirms. |
| valhalla | same | `docker compose restart valhalla-<region>` (per region, sequentially). Entrypoint uses `valhalla_tiles.tar` |
| pelias | same | Restore ES snapshot for the region → switch alias; restart `pelias-<region>-pip`, `pelias-<region>-api`, `pelias-interpolation` |
| geodata | same | `docker compose restart regionservice` |

Activation order for a full data release: **geodata → valhalla → pelias → tiles** (regionservice defines the regions the others must match; tiles are independent and last because they are the largest and least coupled).

`rollback <class>`: flip `current` to the previous release and repeat the same service action. Pelias rollback re-restores the previous snapshot/alias.

Disk safety: `copy` refuses to start if free space on the target drive < release size × 1.1; `prune` keeps 2 releases per class.

### 3.5 What data-manager must provide for this

- Publish step assembling `releases/<id>/<class>/…` plus a `manifest.json` with per-file hashes (also required by tilesservice, which reads `manifest.json`).
- Retention of ≥ 2 planet downloads (`download.tiles_keep=2`).
- The `compose-single-machine` deploy driver wraps this same rsync/verify/switch/activate logic, recorded as a `deployment`; rollback = deploying an older package tag (see `RELEASE-CONTRACT.md` §3–§4).

---

## 4. Migration checklist (ordered)

Each step lists the owner repo and how to verify it.

### Phase A: preparatory fixes (no service impact)

- [ ] **A1 tilesservice PR 0:** fix pre-gzipped pass-through ignoring `Accept-Encoding` (`internal/server/http_tile.go:143-154`). *Verify:* `curl` without `Accept-Encoding: gzip` returns uncompressed MVT; unit test.
- [ ] **A2 swayrider-api:** add `tiles` rate-limit class per user (`RATE_LIMIT_USER_TILES`, default 3000/min as an estimate), raise `MaxIdleConnsPerHost` 10 → 50–100, fix docs (§5.2). *Verify:* tests; load test with glyph/sprite/tile burst.
- [ ] **A3 data-manager styles stage:** several named styles, style version, `manifest.json`, glyph and sprite assets. *Verify:* style preview in the UI; manifest validates.
- [ ] **A4 data-manager package + deploy configuration + deploy (Phase 3):** implement `data-manager/RELEASE-CONTRACT.md` §6 on the data-manager server (packages in `releases/<tag>/` with hash `package.json`, `compose-single-machine` driver, `/repo` and `/deploy` pages); `tiles_keep=2`. *Verify:* package tree matches §3.3; fixture deploy + rollback drill passes locally, then on dev-mini.
- [ ] **A5 data-manager container:** `infra/data-manager/compose.yaml` (dedicated Redis `sw-datamanager-redis`, host port 36389; later Flask + worker). *Verify:* `./debug.sh` against the compose Redis.

### Phase B: infra (dev-mini)

- [ ] **B1** Introduce the per-class roots (§3.2) in `layer-00/10/20 env.example` and compose; mounts point at `<ROOT>` (read-only where possible); drop `TILES_CACHE_PATH`.
- [ ] **B2** Add `pelias-interpolation` to `layer-10/compose.yaml` on `net-sw-dev-pelias`: image from the Pelias interpolation build, `command: ./interpolate server <region>/address.db <region>/street.db`, port 4300 (host port in the 331xx range, one instance per region if DBs are per region), read-only mount of `${PELIAS_ROOT}/current/<region>/interpolation`.
- [ ] **B3** Set `interpolation.client = {adapter: http, host: http://pelias-interpolation:4300}` in each region's API `pelias.json` (the data-manager `pelias` stage currently emits `adapter = null`; extend it or patch at deploy).
- [ ] **B4** Valhalla compose: mount `${VALHALLA_ROOT}/current/<region>` at `/custom_files`; keep `valhalla.json` pointing at `valhalla_tiles.tar`.
- [ ] **B5** PIP mounts: `${PELIAS_ROOT}/current/<region>/wof` → `/data/whosonfirst`. Confirm the same patched SQLite as used for indexing (patched locality ids `9_000_000_000_000 + OSM relation id`).
- [ ] **B6** Write `infra/dev-mini/README.md` (missing today); fix `sw-dev-germany-pip` → `sw-dev-pelias-germany-pip` naming; refresh stale `infra/dev/README.md` regions.
- [ ] **B7** Implement `deploy.sh copy/activate/rollback/list` (§3.4) in `infra/dev-mini/scripts/`; mark legacy `deploy-*.sh` deprecated.
- [ ] **B8** Host prerequisites: `vm.max_map_count >= 262144`; per-drive directories created with correct ownership (ES uid 1000).

### Phase C: tilesservice (see `data-manager/TILESSERVICE-PMTILES.md`)

- [ ] **C1** PR 1: PMTiles v3 reader (single file); validate header `tile_type=MVT`, `tile_compression=gzip`; zoom limits from header (0–15) instead of hard-coded 16; 204 outside range.
- [ ] **C2** PR 2: release holder: `TILES_ROOT/current` symlink resolved at start and on SIGHUP/poll; atomic swap; add `/ready`.
- [ ] **C3** PR 3: routing `/{tileset}/{z}/{x}/{y}` with `base` (legacy MBTiles) and `planet` (PMTiles) side by side; `/{tileset}/tiles.json`.
- [ ] **C4** PR 4: styles from the release (`styles/<id>/<version>/{light,dark}.json`) with `{{.TilesBaseURL}}` and `{{.Tileset}}`, ETag; `/fonts/{fontstack}/{file}`; `/sprites/{file}`.
- [ ] **C5** PR 5 (after cutover): remove `internal/tilecache`, `internal/mbtiles`, `internal/tileindex`, cgo/go-sqlite3 in the Dockerfile.
- [ ] **C6** Compose layer-20: `${TILES_ROOT}:/data/tiles:ro`, env `TILES_ROOT=/data/tiles`.
- [ ] *Verify:* tile/style/glyph/sprite through the gateway; atomic reload while under load; legacy `base` still served.

### Phase D: swayrider-api

- [ ] **D1** No route change (`/v1/tiles/` reverse proxy already forwards new paths and `ETag`/`Cache-Control`/`Content-Encoding`).
- [ ] **D2** Rate-limit class, transport tuning (A2).
- [ ] **D3** Docs: `README.md:315` says tiles need no auth, `API.md:40` lists them as public, `routes.go:57` requires a verified user. Make all three agree; update `openapi.yaml`.
- [ ] **D4** Confirm the service-token refresh fix is deployed.

### Phase E: unaffected services (verify only)

- [ ] **E1 regionservice:** consumes `GEODATA_DIR/manifest.yml`; keep `regions.<name>.contour.{core,extended}` and `shared.border-crossings.<pair>` with `hash`, `hash-type`, `remote-file`. *Verify:* data-manager border output matches against a legacy run on real data.
- [ ] **E2 routerservice:** `VALHALLA_REGION_HOSTS/PORTS` unchanged; region names must match.
- [ ] **E3 searchservice:** `PELIAS_REGIONS` unchanged; verify address and transit results.

### Phase F: dev-mini end-to-end

- [ ] **F1** Configure benelux/france/germany in data-manager; run all stages; approve runs.
- [ ] **F2** Publish release; `deploy.sh copy` for each class (ideally from a different machine than the target).
- [ ] **F3** `activate` in order geodata → valhalla → pelias → tiles.
- [ ] **F4** Smoke tests: tile + style + glyph + sprite via `swayrider-api`; cross-border route; geocode a street, an interpolated house number, a transit stop; regionservice region lookup.
- [ ] **F5** Rollback drill per class; repeat with 2 planets on disk to confirm space.
- [ ] **F6** Record results back into `data-manager/DESIGN.md`.

### Phase G: cutover and retirement

- [ ] **G1** Mobile app switches style/tile source to `planet`.
- [ ] **G2** Remove `base`, MBTiles code, `TILES_CACHE_PATH` (C5).
- [ ] **G3** Delete `infra/dev/scripts/deploy-*.sh`, `infra/DATA_PIPELINE.md`, `swayrider/DATAPIPELINE.md`; archive `data-pipeline/` repo; remove `data-pipeline` from `tools/release.py` (`ORDER`, `DEPS`; add `data-manager`), `tools/README.md:134`, `_github/profile/README.md:75-76`, `data-pipeline/.github`.
- [ ] **G4** Port `dev` (8 regions) from the same model once dev-mini is validated.

---

## 5. Per-service adaptation details

### 5.1 tilesservice

Current state: `TILES_PATH` required; `internal/tileindex/index.go` maps z≤6 → `L0.mbtiles`, z≤10 → `L1/`, z≥11 → `L2/` grid files and merges overlapping cells (`mbtiles.MergeTiles`); `internal/mbtiles/reader.go:34` opens sqlite `?mode=ro`, flips XYZ→TMS; `http_tile.go:75` ignores `{tileset}`; z>16 → 400 (`:95`); caching in `internal/tilecache`.

Target: single PMTiles file per release, no merging, no caches, no cgo. Config: `TILES_ROOT` replaces `TILES_PATH`/`DISK_CACHE_*`/`COMPRESSION_*`; keep `STYLES_PATH` only if styles are not yet shipped in the release. Zoom range: 0–15 (clients over-zoom; verify MapLibre `maxzoom: 15`). Legacy `base` remains until Phase G. Scope `tiles:serve` retained. Details and PR split: `data-manager/TILESSERVICE-PMTILES.md`.

### 5.2 swayrider-api

`internal/handlers/tiles.go` (plain `NewSingleHostReverseProxy`, strips cookies, injects service bearer), route `internal/server/routes.go:57`, rate limit `internal/middleware/ratelimit.go:~111` (currently "public" per-IP class, `RATE_LIMIT_IP_PUBLIC=600`/min), config `TILESSERVICE_HOST/PORT` (`internal/config/config.go:100`). Adaptations: new `tiles` per-user class, higher idle connections per host, doc fixes (D3). No MBTiles/PMTiles/Range references exist.

### 5.3 regionservice

Reads `GEODATA_DIR` (mounted `:ro`). Format and layout fixed by `internal/geodata/manifest.go` (`manifest.yml`), `contour.go`, `border_crossing.go`. data-manager's border stage must keep CSV columns and manifest schema. Mount changes from `${GEODATA_PATH}` to `${GEODATA_ROOT}/current`.

### 5.4 routerservice and Valhalla

No code change. One Valhalla container per region on 33001+, mounting the region directory at `/custom_files`; rename `tiles.tar` → `valhalla_tiles.tar` happens in the release assembly (not at deploy time). `admin.sqlite`/`tz_world.sqlite` come from the same stage; `polylines.0sv.gz` is only an input for Pelias and is not deployed.

### 5.5 Pelias, Elasticsearch, searchservice

- **Index:** `pelias-index-snapshot` per region is copied to `ES_SNAPSHOTS_PATH`, restored (verified on Luxembourg: 69 MB, 2.2 s, 206,203 docs) and the alias switched. Old indices are cleaned afterwards (replaces `clean-es-indices.sh`).
- **PIP:** reads the patched WOF `sqlite/` directory.
- **Interpolation:** new `pelias-interpolation` service (`./interpolate server address.db street.db`, port 4300); API config `interpolation.client` (B3).
- **Placeholder:** remains the approved `placeholder:store` download; mount under `PELIAS_ROOT`.
- **Transit:** GTFS stops are in the index (e.g. Benelux 126,997 stops); API layer config must include the transit source.
- **searchservice:** unchanged (`PELIAS_REGIONS=region=http://pelias-<region>-api:3100/v1`).

---

## 6. dev-mini reference configuration

Illustrative env (`infra/dev-mini/layer-*/env`):

```
TILES_ROOT=/mnt/ssd-a/swayrider/tiles
VALHALLA_ROOT=/mnt/ssd-b/swayrider/valhalla
PELIAS_ROOT=/mnt/ssd-b/swayrider/pelias
GEODATA_ROOT=/mnt/ssd-b/swayrider/geodata
ES_DATA_PATH=/mnt/ssd-c/swayrider/es-data
ES_SNAPSHOTS_PATH=/mnt/ssd-c/swayrider/es-snapshots
```

Compose volume changes (sketch):

```yaml
tilesservice:       { volumes: ["${TILES_ROOT}:/data/tiles:ro"], environment: { TILES_ROOT: /data/tiles } }
regionservice:      { volumes: ["${GEODATA_ROOT}/current:/data/geodata:ro"] }
valhalla-benelux:   { volumes: ["${VALHALLA_ROOT}/current/benelux:/custom_files",
                                "./valhalla/benelux/valhalla.json:/custom_files/valhalla.json:ro"] }
pelias-benelux-pip: { volumes: ["${PELIAS_ROOT}/current/benelux/wof:/data/whosonfirst"] }
pelias-benelux-interpolation:      # NEW
  image: <pelias interpolation image>
  command: ./interpolate server /data/address.db /data/street.db
  volumes: ["${PELIAS_ROOT}/current/benelux/interpolation:/data:ro"]
```

Note: mounting `<ROOT>/current` directly pins the container to the inode at start; if the symlink target changes, a restart is required. For tiles, mount `<ROOT>` and let tilesservice resolve `current` itself (hence no restart). For Valhalla/Pelias/regionservice the restart in §3.4 handles it.

Disk guide for dev-mini (3 regions): tiles 2 × 140 GB; ES data several hundred GB (127–207 M documents per region); build host needs the planet (and 2 versions), per-region PBFs, Pelias importer workspace, interpolation DBs.

---

## 7. Risks and open questions

| Item | Risk | Mitigation |
|---|---|---|
| Protomaps terms | Automated repeated planet downloads and public serving may be restricted | Review terms before Phase F; fall back to self-hosted extract (`pmtiles extract`) |
| Disk | 2 planets ≈ 280 GB + ES + Valhalla | Dedicated SSDs; copy pre-check |
| Silent importer failures | csv-importer/transit exit 0 on rejected config | Judge by documents added; stage fails if count is 0 |
| WOF patch coverage | Spiked only on BE/DE; Bavaria lacks address source | Verify FR/Benelux in Phase F |
| PIP/WOF id alignment | PIP must read the same patched DB used for indexing | One `pelias-wof` asset used by both |
| SQLite concurrency | Several heavy jobs may contend | Limit parallel runs (`run.*` settings) |
| Border/Valhalla unverified | Output may differ from legacy | Compare with a legacy run (E1) |
| `current` symlink + bind mounts | Container keeps old inode | Restart in activate; tiles resolves itself |
| Tiles auth inconsistency | Docs vs code disagree | D3 |

---

## 8. Verification matrix and rollback

| Check | Command / observation | Phase |
|---|---|---|
| Release integrity | all hashes in `manifest.json` verified on target | F2 |
| Tiles | `GET /v1/tiles/planet/{z}/{x}/{y}` 200/204, `tiles.json`, style, glyph, sprite; `/ready` | F4 |
| Tile reload | switch `current` under load; zero 5xx | C2/F5 |
| Routing | cross-border route via routerservice | F4 |
| Geocode | street, interpolated address, transit stop | F4 |
| Regions | regionservice point/bbox/radius lookups, border crossings | F4 |
| Rollback | `deploy.sh rollback <class>` per class returns previous behaviour | F5 |

Rollback is always: flip `current` to the previous release, then repeat the class's activation action (§3.4). Because releases are immutable and the previous two are retained, no rebuild is needed.
