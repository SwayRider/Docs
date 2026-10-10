# Migration: data-pipeline → data-manager

Status: **first deploy to dev-mini done** (plan written 2026-10-05, status updated 2026-10-10; see [Status 2026-10-10](#status-2026-10-10)). Owner: platform. Test-bed: `infra/dev-mini` (single-server docker-compose).

`data-manager/` replaces the Python scripts in `data-pipeline/`. This document is the single place that lists every step needed to move SwayRider over: what changes in each service, in infra, how data reaches a target server, and the order to do it in.

Related documents:

- [`data-manager/CLAUDE.md`](../data-manager/CLAUDE.md), [`DESIGN.md`](../data-manager/DESIGN.md), [`SERVICES.md`](../data-manager/SERVICES.md): data-manager design and what it expects from each downstream service.
- [`data-manager/TILESSERVICE-PMTILES.md`](../data-manager/TILESSERVICE-PMTILES.md): detailed tilesservice handover (PR 0–5).
- [`data-manager/RELEASE-CONTRACT.md`](../data-manager/RELEASE-CONTRACT.md) (package/deploy contract and as-built notes), [`data-manager/DEPLOY-DEV-MINI.md`](../data-manager/DEPLOY-DEV-MINI.md) (runbook of the first deploy), [`data-manager/README.md`](../data-manager/README.md) (prerequisites, how to start; debug build only).
- [`TECHNOLOGY_DECISIONS.md`](TECHNOLOGY_DECISIONS.md): stack rationale (updated for this migration).

---

## Status 2026-10-10

Releases (tags on `main`): data-manager v0.0.1 (a debug build, see its README), infra v0.2.0, tilesservice v0.2.0, regionservice v0.2.0. `swayrider-api` is at v0.1.6 plus 9 commits on `main` in this checkout; whether a newer release was cut is not verified here.

A real deploy to dev-mini has been done (same machine, driver `compose-single-machine`, tiles through Garage/S3). Package `r-20261007-1` (357 GB) was deployed for geodata, valhalla, pelias and tiles; pelias was redeployed twice afterwards (`r-20261009-1`, `r-20261010-1`) after two fixes (below). The target keeps only `current` and `previous`.

**Verified on the real deploy**

- tiles: 138.5 GB multipart upload to Garage (`releases/<tag>/`, `current.json` written last); tilesservice (image with the PMTiles reader, on `net-sw-dev-data`) opens `s3://swayrider-tiles/releases/<tag>/tiles.pmtiles`; Garage logs its reads with the read-only key.
- pelias: restore from the ES snapshot, indices pinned in `pelias.json` (`api.indexName`); street search works and house-number interpolation was verified for benelux (`match_type: interpolated`).
- regionservice (`manifest.yml`), routerservice and searchservice: only "container healthy / startup log OK".

**Not verified yet**

- rollback drill and failure drill (`data-manager/DEPLOY-DEV-MINI.md` §6);
- an authenticated tile/style request through `swayrider-api`;
- functional calls to regionservice and routerservice;
- pelias search for France and Germany;
- reload of a new tiles release without a restart (tilesservice does not reload on `current.json` yet: C2).

**Two bugs found in data-manager by the real deploy (both fixed there):** the polylines importer reads plain text (the `.gz` edge file produced garbage street documents; it is now gunzipped), and the API `pelias.json` lacked `api.services.interpolation` (see B3).

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
| Distribution | Tagged tarballs in a geodata store + `infra/*/scripts/deploy-*.sh` extract them | Tagged **packages** in a package repository, **copied** to the target by a deploy and activated with a `current` symlink (tiles: object store); see §3 |
| Machine API | n/a | None (UI and `flask` CLI). Package repository and deploy are built (`/repo`, `/deploy`, `flask deploy-*`) |

### 1.2 Artifact flow

```
 BUILD HOST (data-manager; may be a different machine)          TARGET HOST (dev-mini, docker compose)
 ┌───────────────────────────────────────────────┐              ┌──────────────────────────────────────────┐
 │ DATA_ROOT/                                    │              │ TILES_ROOT/     current -> releases/<id>  │──► tilesservice (ro)
 │  downloads/  library/assets/  work/  tools/   │   copy+hash  │ VALHALLA_ROOT/  current -> releases/<id>  │──► valhalla-<region> (ro)
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
| Pelias ES snapshot / WOF / config | Elasticsearch, pelias-pip, pelias-api → searchservice | Snapshot restore into a pinned index (no alias switch); PIP reads patched WOF; API config updated | §5.5 |
| Interpolation DBs | **new** `pelias-<region>-interpolation` service | New compose service (`PORT=4300`) + `api.services.interpolation.url` in API `pelias.json` | §5.5 |
| Everything | infra/dev-mini | New env vars/roots, mounts, new service, Garage, README (a data-manager compose for Redis is still planned) | §3, §6 |

---

## 2. Current status and gaps

Built in data-manager: configuration UI, downloads, OSM, border, Valhalla, styles (a template style with an id and an integer version, glyphs and sprites, `manifest.json`), Pelias (+ WOF patch, Overture/GTFS, interpolation), tiles download, the package repository (`/repo`, `flask package-*`, cleanup after packaging) and the deploy (`datamanager/deploy/`, `/deploy`, `flask deploy-*`). Per-item state of the migration: §4. What the first real deploy showed: [Status 2026-10-10](#status-2026-10-10).

**Still open before cutover:**

1. **tilesservice** does not reload on `current.json` yet, serves styles from `STYLES_PATH` (not from the release) and has no `/ready`, `tiles.json`, `/fonts` or `/sprites` routes (C2–C4). The data-manager deploy compensates by recreating the container with a new `PMTILES_URL`.
2. **swayrider-api** has no `tiles` rate-limit class, still uses `MaxIdleConnsPerHost: 10`, and its README and API.md disagree with `routes.go` about tile authentication (A2, D3).
3. **Rollback is unproven:** neither the rollback drill nor the failure drill has run (F5).
4. **data-manager is a debug build:** no release build, no container image, the development server has no authentication. The dedicated Redis compose (`infra/data-manager/compose.yaml`) is referenced by the data-manager README but is not in the `infra` repo (A5).
5. **Unverified:** Pelias for France and Germany, regionservice/routerservice behaviour beyond a clean start, border stage output vs a legacy run on real data, Protomaps terms for repeated automated downloads and public serving, MapLibre `maxzoom: 15` over-zoom behaviour; WOF patch spiked on Belgium and Germany.
6. **Known hazards:** csv-importer/transit importers exit 0 after rejecting config (judge by documents added); SQLite concurrency with several heavy jobs unconfirmed; ES needs `vm.max_map_count >= 262144` (checked by `prepare-host.sh`).

---

## 3. Deployment strategy (dev-mini as single-server test-bed)

### 3.1 Principles

- **data-manager builds and packages releases; the target needs none of its tools.** Deploying is always a *copy* (resumable, checksum-verified). Built so far: a local target (data-manager on the target machine, as on dev-mini; the driver rejects ssh for now) and the S3 transport for tiles. A target host never needs Python, osmium, tippecanoe or GDAL.
- **One root per artifact class**, each configured by an env var so it can sit on its own drive (tiles ≈ 140 GB × 2 on a large SSD; ES data on the fastest; Valhalla/Pelias/geodata elsewhere).
- **Immutable releases + atomic switch:** `<ROOT>/releases/<tag>/…` (the release id is the package tag, `r-YYYYMMDD-N`) plus a relative symlink `<ROOT>/current -> releases/<tag>`. A copy lands in `<tag>.partial` and is renamed when verified, so services never see a half-copied release.
- **A target keeps only `current` and `previous`** per class (rollback target). After a healthy deploy every other release and leftover `.partial` is removed; a rollback removes the release that was rolled back. `deploy --drop-previous` removes `previous` before the copy to save space, at the price of having no rollback target until the deploy is healthy.
- **Retention in the package repository is manual.** Nothing deletes packages by itself; labels (`v1.0.1`, `test-…`) are added, changed and removed per label; a package that is live on a target (label `live=<config>`) cannot be deleted. `packages-prune` is an explicit CLI command.
- **Activation is per class** (container recreate/restart, snapshot restore, env file + recreate for tiles) and part of the same deploy; it runs after the class's copy has been verified.

### 3.1a Tiles live in an object store (decision 2026-10-05)

The **tiles** class does not use a copied directory. The planet release (`tiles.pmtiles`, `manifest.json`, styles, glyphs, sprites) is uploaded to an S3-compatible object store (**Garage**, added to `infra/dev-mini/layer-00`), and `tilesservice` reads it with ranged GETs. This reverses the earlier "no object store" decision (one file on one host). Reasons: no 140 GB × 2 copy on every tilesservice host, one shared copy for replicas/k8s later, a laptop can point at a remote planet. The other classes (valhalla, pelias, geodata) stay copy-based as described below.

- Bucket `swayrider-tiles`: `releases/<id>/…` (same tree as §3.3) plus a small pointer object `current.json` (`{"schema":1,"release":"<id>","prefix":"releases/<id>/","updated_at":"…"}`) that replaces the `current` symlink; a single PUT switches releases atomically and is written **last**. The target design is that `tilesservice` polls the pointer and reloads with no activation step (C2, **not built yet**). Until then the deploy writes `infra/dev-mini/layer-20/tiles-release.env` (`PMTILES_URL=s3://swayrider-tiles/releases/<tag>/tiles.pmtiles`) and runs `docker compose up -d --no-deps --force-recreate tilesservice` (a plain restart does not re-read `env_file`); `tilesservice` needs `net-sw-dev-data` to reach `sw-dev-garage`.
- Two keys, from env/secret stores only (never in packages or the deploy configuration): read/write for the data-manager deploy, read-only for `tilesservice`.
- The data-manager deploy for this class is the `s3` transport of the `compose-single-machine` driver (multipart upload, verify, write `current.json` last, `previous.json` keeps the release before; only the `current` and `previous` prefixes stay): see `data-manager/RELEASE-CONTRACT.md` §3.3 and the as-built notes in §9. Deployed to dev-mini (138.5 GB) and opened by tilesservice.
- `tilesservice` keeps a local-file backend (`file`) for tests, laptops with a small extract and a single host that does not want the store. Reader design: `io.ReaderAt` with a `file` and an `s3` implementation (`data-manager/TILESSERVICE-PMTILES.md`, update note at the top).
- Garage and the credentials are infra work (Phase B, **B0**, built in `infra/dev-mini/layer-00`).

### 3.2 Per-class roots (env vars, `infra/dev-mini`)

| Variable | Example mount | Content | Size guide | Notes |
|---|---|---|---|---|
| `TILES_ROOT` | `/mnt/ssd-a/swayrider/tiles` | legacy `base/` MBTiles during transition; PMTiles releases only with the `file` backend (§3.1a) | ~140 GB × 2 (`file` backend) | Replaces `TILES_DATA_PATH`. `TILES_CACHE_PATH` still exists in dev-mini for the legacy `base` cache; removed in Phase G |
| `GARAGE_DATA_PATH` / `GARAGE_META_PATH` | `/mnt/ssd-a/swayrider/garage/data` / `…/meta` | Object store for the PMTiles releases (§3.1a) | ~140 GB × 2 data; small metadata on a fast disk | New (Phase B, B0) |
| `VALHALLA_ROOT` | `/mnt/ssd-b/swayrider/valhalla` | per-region tiles, admin/tz sqlite | tens of GB | Replaces `VALHALLA_DATA_PATH` |
| `PELIAS_ROOT` | `/mnt/ssd-b/swayrider/pelias` | WOF sqlite, interpolation DBs, `pelias.json`, placeholder | tens of GB | Replaces `PELIAS_DATA_PATH` |
| `GEODATA_ROOT` | `/mnt/ssd-b/swayrider/geodata` | `manifest.yml`, contours, border-crossings | < 1 GB | Replaces `GEODATA_PATH` |
| `ES_SNAPSHOTS_PATH` | `/mnt/ssd-c/swayrider/es-snapshots` | snapshot repo (existing var) | per-region snapshots | Restore source |
| `ES_DATA_PATH` | `/mnt/ssd-c/swayrider/es-data` | live ES data (existing var) | 100s of GB for 3 regions | Fastest drive |
| `DATA_ROOT` (build host only) | `/data/swayrider-dm` | data-manager working set | large | Not on the target |

### 3.3 Folder structure

Target host (example for dev-mini: benelux, france, germany). The `tiles` tree below is the layout of a release; with the object store (§3.1a) it is the key layout under `releases/<id>/` in the bucket instead of a directory under `$TILES_ROOT`, and `current` is the `current.json` object:

```
$TILES_ROOT/
  current -> releases/r-20261012-1
  releases/
    r-20261012-1/
      tiles.pmtiles
      manifest.json                 # generated at packaging
      styles/<style-id>/<version>/{light,dark}.json
      glyphs/<fontstack>/<range>.pbf
      sprites/<name>[@2x].{json,png}
    r-20261005-1/                   # previous (rollback); nothing older is kept
  base/                             # legacy MBTiles L0.mbtiles, L1/, L2/ (transition only, removed in Phase G)

$VALHALLA_ROOT/
  current -> releases/<tag>
  releases/<tag>/<region>/{valhalla_tiles.tar, admin.sqlite, tz_world.sqlite}   # region ∈ benelux|france|germany
  work/<region>/                    # scratch directory mounted as /custom_files (uid 59999)

$PELIAS_ROOT/
  current -> releases/<tag>
  releases/<tag>/
    placeholder/data/store.sqlite3  # Placeholder store (part of the pelias class)
    <region>/wof/sqlite/            # patched WOF (wof.tar.gz unpacked), read by pelias-pip
    <region>/interpolation/{street.db,address.db}
    <region>/pelias.json            # production config: pins the ES index (api.indexName), api.services.interpolation.url

$GEODATA_ROOT/
  current -> releases/<tag>
  releases/<tag>/{manifest.yml, contours/<region>-{core|extended}.geojson, border-crossings/<a>-<b>.csv}
                                    # manifest.yml is generated by data-manager at packaging (legacy shape);
                                    # border-crossing files keep single-dash names (a-b.csv, not a--b)

$ES_SNAPSHOTS_PATH/<tag>/<region>/…   # unpacked from the package only for the restore, removed once the release is live
```

Build host: `DATA_ROOT` (from `datamanager/config.py`) holds `state/`, `downloads/`, `library/`, `work/`, `tools/`, `deploy-state/`; the packages live in the package repository `PACKAGE_ROOT` (default `DATA_ROOT/releases`; on dev-mini a shared directory owned by `root:swdata`, setgid, default ACL so every administrator can package and deploy: `data-manager/scripts/init-host.sh` and `infra/dev-mini/scripts/prepare-host.sh` set it up).

Region names must be identical across `GEODATA_ROOT` manifest, `VALHALLA_REGION_*`, `PELIAS_REGIONS` and the directory names above.

### 3.4 Copy + activate procedure (as built)

Data-manager models this as **package** (tagged archive) + **deploy configuration** (pluggable driver; first `compose-single-machine`) = **deploy**; contract and as-built notes in [`data-manager/RELEASE-CONTRACT.md`](../data-manager/RELEASE-CONTRACT.md) (§3, §9), the first-deploy runbook in [`data-manager/DEPLOY-DEV-MINI.md`](../data-manager/DEPLOY-DEV-MINI.md). Commands: `flask deploy-plan`, `flask deploy [--classes …] [--drop-previous]`, `flask deploy-state`, `flask deploy-rollback`, or the `/deploy` page.

Per class the driver:

1. checks free space on the target (size × 1.1);
2. copies into `<ROOT>/releases/<tag>.partial/` (resumable through `.deploy.json`; the source is hashed while copying/unpacking);
3. verifies against `package.json` (`deploy.verify`: full sha256 or size);
4. renames to `<tag>`, flips the relative `current` and `previous` links;
5. activates and health-checks the class; on failure it switches back;
6. prunes everything except `current` and `previous`.

Activation (an activator per class, configured in the deploy configuration):

| Class | Switch | Service action |
|---|---|---|
| geodata, valhalla | `current` link | `docker compose up -d --no-deps --force-recreate <service>` per region (the compose files do not start a service whose data directory is missing, so the first deploy has to create the containers; with no `compose_file` it is a `docker restart`) |
| pelias | `current` link | Per region: read `schema.indexName` from the release's `pelias.json`; if the index does not exist, register the snapshot repository `dm_<tag>_<region>`, restore, then restart pip/interpolation/api (+ Placeholder). **No alias switch:** `pelias.json` pins the concrete index `pelias_<region>-<run>`. Unpacked snapshots and repositories are removed once the release is healthy; indices of removed releases are deleted (except one a kept release still pins) |
| tiles | upload to Garage, `current.json` written last (`previous.json` = the release before) | write `tiles-release.env` (`PMTILES_URL=s3://…`) and recreate `tilesservice` (see §3.1a) |

Supporting services are managed by the deploy as well: `ensure` (`authservice`, `swayrider-api-register`, `mailservice`, `pelias-libpostal`: started, never recreated) and `ensure_after` (`routerservice` after valhalla, `searchservice` after pelias, started once the class is healthy). A supporting service that does not start is a warning in the run report, not a failed deploy.

Activation order for a full data release (`activation_order`): **geodata → valhalla → pelias → tiles** (regionservice defines the regions the others must match; tiles are independent and last because they are the largest and least coupled). A failed class stops the sequence; the classes before it stay live and a re-run resumes.

**Rollback** (`flask deploy-rollback`) is a new deployment that switches `current` back to `previous`, repeats the class's activation and then **removes the release that was rolled back** (pelias: its index, unless the old release pins the same one); `previous` is empty afterwards. Pelias rollback needs no snapshot because the restored indices stay in Elasticsearch. Rollback has **not been exercised** yet (F5).

Disk safety: besides the space check, the target holds three releases for a while on every deploy (the old `previous` is only removed after the new release is healthy); for tiles in Garage and pelias indices in ES that is a lot, hence `--drop-previous`.

**Manual fallback:** `infra/dev-mini/scripts/release.py` (`copy`, `activate`, `rollback`, `list`, `prune`, `es-restore`; valhalla, pelias, geodata) does the same by hand with the same layout. The legacy `deploy.sh` and `dev/scripts/deploy-*.sh` (data-pipeline tarballs) are deprecated.

### 3.5 What data-manager provides for this

- Packaging: `releases/<tag>/<class>/…` plus `package.json` with per-file hashes, and the generated parts (tiles `manifest.json` read by tilesservice, geodata `manifest.yml`, the Placeholder store, the snapshot restore names). Built.
- Retention of ≥ 2 planet downloads (`download.tiles_keep`, default 2). Built.
- The `compose-single-machine` deploy driver with the copy, verify, switch, activate and rollback logic of §3.4, recorded as a `deployment`; rollback is `flask deploy-rollback` (or deploying an older package tag).

---

## 4. Migration checklist (ordered)

Each step lists the owner repo and how to verify it. Legend: `[x]` done and checked against the code of the repo it belongs to (2026-10-10), `[~]` partly done (the note says what is missing), `[ ]` not done or not verified. Facts about the real deploy on dev-mini are as reported by the operator; they cannot be checked from the repositories (see [Status 2026-10-10](#status-2026-10-10)).

### Phase A: preparatory fixes (no service impact)

- [x] **A1 tilesservice PR 0:** fix pre-gzipped pass-through ignoring `Accept-Encoding`. Commit `86194f3`, in v0.2.0; `internal/server/encoding.go` (`acceptsGzip`), `Vary: Accept-Encoding`, test in `http_tile_test.go` (no `Accept-Encoding` → raw bytes).
- [ ] **A2 swayrider-api:** add `tiles` rate-limit class per user (`RATE_LIMIT_USER_TILES`, default 3000/min as an estimate), raise `MaxIdleConnsPerHost` 10 → 50–100, fix docs (§5.2). *Not done:* `internal/middleware/ratelimit.go:112` still counts `/v1/tiles/` in the public per-IP class and `internal/handlers/proxy.go` still has `MaxIdleConnsPerHost: 10`. *Verify:* tests; load test with glyph/sprite/tile burst.
- [~] **A3 data-manager styles stage:** the stage writes a template style with an id, a name and an integer version; the tiles class packages `styles/<id>/<version>/{light,dark}.json`, glyphs, sprite sheets and a generated `manifest.json` (`stages/styles.py`, `services/packages.py: tiles_manifest`). *Missing:* several named styles (one style per configuration for now).
- [x] **A4 data-manager package + deploy configuration + deploy (Phase 3):** `RELEASE-CONTRACT.md` implemented: packages in the package repository with hash `package.json`, `compose-single-machine` driver with activators, S3 tiles transport, `/repo` and `/deploy` pages, `flask package-*`/`deploy-*`; `download.tiles_keep` defaults to 2. Ran for real on dev-mini (package `r-20261007-1`). *Not done:* the rollback drill on dev-mini (F5); the fixture tests are `tests/test_deploy_*.py` and `tests/test_package*.py` (not re-run for this update).
- [ ] **A5 data-manager container:** `infra/data-manager/compose.yaml` (dedicated Redis `sw-datamanager-redis`, host port 36389; later Flask + worker). *Not verified:* the file is not in the `infra` repo (v0.2.0 / `main`) and `infra/README.md` calls it "planned"; the data-manager README and CLAUDE.md already refer to it. There is no data-manager container image (debug build only).

### Phase B: infra (dev-mini)

- [x] **B0** Garage object store in `infra/dev-mini/layer-00` (`garage/garage.toml`, `garage/init.sh` single-node layout, bucket `swayrider-tiles`, read/write and read-only keys from env; `garage/smoke-test.sh`; data/meta volumes `GARAGE_DATA_PATH`/`GARAGE_META_PATH`; S3 endpoint bind `GARAGE_S3_BIND`). Used for the 138.5 GB tiles upload. Read-only key observed in Garage's log for tilesservice reads.
- [x] **B1** Per-class roots (§3.2) in `layer-00/10/20 env.example` and compose; mounts point at `<ROOT>` (read-only, `create_host_path: false`). *Note:* `TILES_CACHE_PATH` is still in `layer-20` for the legacy `base` cache; it goes in Phase G.
- [x] **B2** `pelias-<region>-interpolation` in `layer-10/compose.yaml` on `net-sw-dev-pelias`: image `pelias/interpolation`, image CMD `./interpolate server /data/interpolation/address.db /data/interpolation/street.db`, **`PORT=4300`** (the server listens on `$PORT`, default 3000), host ports 33112/33122/33132, read-only mount of `${PELIAS_ROOT}/current/<region>/interpolation` at `/data/interpolation`.
- [x] **B3 (corrected 2026-10-10)** The Pelias API reads **`api.services.interpolation.url`** (e.g. `http://pelias-benelux-interpolation:4300`). `interpolation.client` is only used by the importers and stays `{adapter: "null"}`. The earlier plan (`interpolation.client = {adapter: http, …}`) was wrong. data-manager emits the URL for the production `pelias.json` (`services/pelias_data.py`, commit `7ed3226`); house-number interpolation was verified for benelux (`match_type: interpolated`).
- [x] **B4** Valhalla compose: `${VALHALLA_ROOT}/work/<region>` is the scratch directory mounted at `/custom_files`; the release's `valhalla_tiles.tar`, `admin.sqlite` and `tz_world.sqlite` (from `${VALHALLA_ROOT}/current/<region>`) are mounted read-only into it; `valhalla.json` points at `/custom_files/admin.sqlite` and `/custom_files/tz_world.sqlite`.
- [x] **B5** PIP mounts: `${PELIAS_ROOT}/current/<region>/wof` → `/data/whosonfirst` (`layer-10/compose.yaml`). The patched WOF comes from the same `pelias-wof` asset used for indexing (`wof.tar.gz` is unpacked into `<region>/wof/` by the deploy).
- [x] **B6** `infra/dev-mini/README.md` written; dev-mini uses `sw-dev-pelias-<region>-pip` names (`pelias-germany-pip`). `infra/dev/README.md` carries no region list.
- [x] **B7** Deploy tooling in `infra/dev-mini/scripts/`: `release.py` (`copy`/`activate`/`rollback`/`list`/`prune`/`es-restore`) instead of the planned `deploy.sh`, next to the data-manager deploy that supersedes it; the legacy `deploy*.sh` are marked deprecated.
- [x] **B8** Host prerequisites: `scripts/prepare-host.sh` (`--dry-run` default, `--apply` asks A/M/Q) checks `vm.max_map_count >= 262144`, creates the shared per-class directories (`root:swdata`, setgid, default ACL) and gives the Elasticsearch data and Valhalla scratch directories to their service uid.

### Phase C: tilesservice (see `data-manager/TILESSERVICE-PMTILES.md`)

- [x] **C1** PR 1: PMTiles v3 reader over `io.ReaderAt` (own implementation in `internal/pmtiles`, `file` and `s3` backends in `internal/objstore`); `reader.go` rejects anything but `tile_type=MVT`; `http_planet.go` takes the zoom limits from the header and returns 204 outside the range. v0.2.0; `pmtiles-probe` checks archives and store access.
- [ ] **C2** PR 2: release holder: `current.json` (object store) or the `current` symlink (`file` backend) resolved at start and by polling (SIGHUP for `file`); manifest read from the same place; atomic swap; add `/ready`. *Not built:* the archive is opened once from `PMTILES_URL`; no polling, no `/ready` (only `/v1/tiles/ping`). The data-manager deploy recreates the container instead.
- [~] **C3** PR 3: `GET /v1/tiles/{tileset}/{z}/{x}/{y}` serves `planet` from the PMTiles archive next to the legacy MBTiles (`base`). *Missing:* `/{tileset}/tiles.json`.
- [ ] **C4** PR 4: styles from the release (`styles/<id>/<version>/{light,dark}.json`) with `{{.TilesBaseURL}}` and `{{.Tileset}}`, ETag; `/fonts/{fontstack}/{file}`; `/sprites/{file}`. *Not built:* styles still come from `STYLES_PATH` (`/v1/tiles/styles[/{name}]`, only `{{.TilesBaseURL}}`); there are no font or sprite routes.
- [ ] **C5** PR 5 (after cutover): remove `internal/tilecache`, `internal/mbtiles`, `internal/tileindex`, cgo/go-sqlite3 in the Dockerfile. Still present on purpose.
- [x] **C6** Compose layer-20: `S3_ENDPOINT`, `S3_REGION`, read-only key and `PMTILES_URL` (from `tiles-release.env`, written by the deploy) for `tilesservice`, on `net-sw-dev-data`; legacy `base` keeps `${TILES_ROOT}/base:/data/tiles:ro` until Phase G.
- [ ] *Verify:* tile/style/glyph/sprite through the gateway; atomic reload while under load; legacy `base` still served. *Not verified.*

### Phase D: swayrider-api

- [x] **D1** No route change: `/v1/tiles/` is a plain `httputil` reverse proxy (`internal/handlers/tiles.go`, `proxy.go`) that forwards the path and the upstream response headers. Read from the code; not exercised with an authenticated request.
- [ ] **D2** Rate-limit class, transport tuning (A2). Not done.
- [ ] **D3** Docs: `README.md:315` says tiles need no auth (`README.md:239` says access token), `API.md:40` lists them as public (rate-limit class), `routes.go:57` requires a verified user (`RequireVerifiedUser`). Still disagree; `openapi.yaml` not checked.
- [~] **D4** Service-token refresh fix: in the code (`615a7c2`, bounded refresh timeout, in v0.1.6; `review/CODE_REVIEW_2026-08.md` #4 FIXED 2026-08-17). *Not verified:* that the running dev instance has it.

### Phase E: unaffected services (verify only)

- [~] **E1 regionservice:** consumes `GEODATA_DIR/manifest.yml`; the generated manifest keeps `regions.<name>.contour.{core,extended}` and `shared.border-crossings.<pair>` with `hash`, `hash-type`, `remote-file` (matches `regionservice/internal/geodata/manifest.go`); border-crossing files are named `<a>-<b>.csv`. Deployed and the service started healthy. *Not verified:* region lookups, and data-manager border output vs a legacy run on real data.
- [~] **E2 routerservice:** `VALHALLA_REGION_HOSTS/PORTS` unchanged; started by the deploy (`ensure_after`) and healthy. Functional routing (incl. cross-border) not verified.
- [~] **E3 searchservice:** `PELIAS_REGIONS` unchanged; started by the deploy and healthy. Address and transit results through searchservice not verified.

### Phase F: dev-mini end-to-end

- [x] **F1** Configure benelux/france/germany in data-manager; run all stages; approve runs. Package `r-20261007-1` exists and was deployed.
- [x] **F2** Package (357 GB) and copy for each class, by the data-manager deploy (not `deploy.sh`), from the package repository on the same machine. *Not done:* from a different machine (ssh target not supported yet).
- [x] **F3** Activate in order geodata → valhalla → pelias → tiles (`activation_order`); pelias redeployed twice (`r-20261009-1`, `r-20261010-1`) after the two fixes.
- [~] **F4** Smoke tests. Done: pelias street search; interpolated house number (benelux); tilesservice opens the planet archive in Garage. *Not done:* tile + style + glyph + sprite via `swayrider-api`, cross-border route, geocode a transit stop, regionservice region lookup, pelias search for France and Germany.
- [ ] **F5** Rollback drill per class (and the failure drill: Garage down during the tiles upload); repeat with 2 planets on disk to confirm space. Runbook: `data-manager/DEPLOY-DEV-MINI.md` §6.
- [ ] **F6** Record results back into `data-manager/DESIGN.md` (after F5).

### Phase G: cutover and retirement

Nothing started on purpose: this follows the mobile app move to `planet` and the rollback drill (F5).

- [ ] **G1** Mobile app switches style/tile source to `planet`.
- [ ] **G2** Remove `base`, MBTiles code, `TILES_CACHE_PATH` (C5).
- [ ] **G3** Delete `infra/dev/scripts/deploy-*.sh` (and, with them, `lib.sh`, `clean-es-indices.sh`, `fix-border-tar.sh`, `infra/dev-mini/scripts/deploy.sh`), `infra/DATA_PIPELINE.md`, `swayrider/DATAPIPELINE.md`; archive `data-pipeline/` repo; remove `data-pipeline` from `tools/release.py` (docstring line 10, `ORDER` line 45, `DEPS` line 56; add `data-manager`), `tools/README.md:134`, `_github/profile/README.md:75-76`, `data-pipeline/.github`. Also: `swayrider/README.md` lines 27, 103, 138, 276, 301 and the section from line 402 on, `swayrider/DEVELOPMENT.md:142`, `infra/README.md` legacy tile-layer sections and the "Deprecated" callouts, `Docs/SideTasks` path notes (history, keep).
- [ ] **G4** Port `dev` (8 regions) from the same model once dev-mini is validated.

---

## 5. Per-service adaptation details

### 5.1 tilesservice

Current state: `TILES_PATH` required; `internal/tileindex/index.go` maps z≤6 → `L0.mbtiles`, z≤10 → `L1/`, z≥11 → `L2/` grid files and merges overlapping cells (`mbtiles.MergeTiles`); `internal/mbtiles/reader.go:34` opens sqlite `?mode=ro`, flips XYZ→TMS; `http_tile.go:75` ignores `{tileset}`; z>16 → 400 (`:95`); caching in `internal/tilecache`.

Target: single PMTiles file per release, no merging, no caches, no cgo. Source: the object store (`s3`, decision §3.1a) or a local directory (`file`); manifest, styles, glyphs and sprites come from the same release prefix, the small assets cached in memory. Config (names provisional, fixed in the PRs): `PMTILES_URL` in PR 1, then a source base (`s3://swayrider-tiles` or `file:///data/tiles`) with `S3_ENDPOINT`, `S3_REGION`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`; these replace `TILES_PATH`/`DISK_CACHE_*`/`COMPRESSION_*`; keep `STYLES_PATH` only if styles are not yet shipped in the release. Zoom range: 0–15 (clients over-zoom; verify MapLibre `maxzoom: 15`). Legacy `base` remains until Phase G. Scope `tiles:serve` retained. Details and PR split: `data-manager/TILESSERVICE-PMTILES.md`.

### 5.2 swayrider-api

`internal/handlers/tiles.go` (plain `NewSingleHostReverseProxy`, strips cookies, injects service bearer), route `internal/server/routes.go:57`, rate limit `internal/middleware/ratelimit.go:~111` (currently "public" per-IP class, `RATE_LIMIT_IP_PUBLIC=600`/min), config `TILESSERVICE_HOST/PORT` (`internal/config/config.go:100`). Adaptations: new `tiles` per-user class, higher idle connections per host, doc fixes (D3). No MBTiles/PMTiles/Range references exist.

### 5.3 regionservice

Reads `GEODATA_DIR` (mounted `:ro`). Format and layout fixed by `internal/geodata/manifest.go` (`manifest.yml`), `contour.go`, `border_crossing.go`. data-manager's border stage must keep CSV columns and manifest schema. Mount changes from `${GEODATA_PATH}` to `${GEODATA_ROOT}/current`.

### 5.4 routerservice and Valhalla

No code change. One Valhalla container per region on 33001+, mounting the region directory at `/custom_files`; rename `tiles.tar` → `valhalla_tiles.tar` happens in the release assembly (not at deploy time). `admin.sqlite`/`tz_world.sqlite` come from the same stage; `polylines.0sv.gz` is only an input for Pelias and is not deployed.

### 5.5 Pelias, Elasticsearch, searchservice

- **Index:** `pelias-index-snapshot` per region is unpacked from the package into `ES_SNAPSHOTS_PATH/<tag>/<region>`, registered as repository `dm_<tag>_<region>` and restored (verified on Luxembourg: 69 MB, 2.2 s, 206,203 docs). There is **no alias switch**: `pelias.json` pins the concrete index (`pelias_<region>-<run>`, `api.indexName`). The unpacked snapshots and the repositories are removed once the release is live; indices of removed releases are deleted (replaces `clean-es-indices.sh`).
- **PIP:** reads the patched WOF `sqlite/` directory (`wof.tar.gz` of the package, unpacked into `<region>/wof/`).
- **Interpolation:** new `pelias-<region>-interpolation` service (`./interpolate server`, listens on `$PORT`, infra sets `PORT=4300`; databases at `/data/interpolation/{address,street}.db`); the API reads `api.services.interpolation.url` (B3, corrected: not `interpolation.client`).
- **Placeholder:** the approved `placeholder:store` download; part of the pelias class (`placeholder/data/store.sqlite3`) and mounted from `PELIAS_ROOT`.
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

Compose volume changes (sketch of what `infra/dev-mini` does; the compose files are the reference):

```yaml
tilesservice:       { environment: { S3_ENDPOINT: …, S3_ACCESS_KEY_ID: …, S3_SECRET_ACCESS_KEY: … /* read-only key, planet from Garage, §3.1a */ },
                      env_file: ["./tiles-release.env"] /* PMTILES_URL, written by the deploy */,
                      volumes: ["${TILES_ROOT}/base:/data/tiles:ro"] /* legacy base only */, networks: [net-sw-dev-data] }
regionservice:      { volumes: ["${GEODATA_ROOT}/current:/data/geodata:ro"] }
valhalla-benelux:   { volumes: ["${VALHALLA_ROOT}/work/benelux:/custom_files",                       # scratch
                                "${VALHALLA_ROOT}/current/benelux/valhalla_tiles.tar:/custom_files/valhalla_tiles.tar:ro",
                                "${VALHALLA_ROOT}/current/benelux/admin.sqlite:/custom_files/admin.sqlite:ro",
                                "${VALHALLA_ROOT}/current/benelux/tz_world.sqlite:/custom_files/tz_world.sqlite:ro",
                                "./valhalla/benelux/valhalla.json:/custom_files/valhalla.json:ro"] }
pelias-benelux-pip: { volumes: ["${PELIAS_ROOT}/current/benelux/wof:/data/whosonfirst"] }
pelias-benelux-interpolation:      # NEW
  image: pelias/interpolation
  environment: ["PORT=4300"]       # the server listens on $PORT (default 3000)
  volumes: ["${PELIAS_ROOT}/current/benelux/interpolation:/data/interpolation:ro"]   # address.db, street.db
```

Note: mounting `<ROOT>/current` directly pins the container to the inode at start, and a plain restart does not re-read an `env_file`. The deploy therefore creates or recreates the services of a class (`docker compose up -d --no-deps --force-recreate`), which also covers the first deploy when the containers do not exist yet (the compose files do not start a service whose data directory is missing). Once tilesservice reloads on `current.json` (C2) the tiles recreate goes away.

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
| `current` symlink + bind mounts | Container keeps old inode | The deploy recreates the class's services; tiles: recreate with a new `PMTILES_URL` until C2 |
| Tiles auth inconsistency | Docs vs code disagree | D3 |
| Object store latency | Every uncached tile is a ranged GET (about one round trip) instead of a page-cached local read | Directory cache in memory; measure on dev-mini before adding a tile cache |
| Object store credentials/availability | A new service holds the planet; wrong key scopes or a down store stops the map | Read-only key for tilesservice, separate deploy key, health check in `/ready`; `file` backend as fallback |
| Upload of ~140 GB | Slow or interrupted multipart upload | Resumable per-object upload, verify before writing `current.json` |

---

## 8. Verification matrix and rollback

| Check | Command / observation | Phase |
|---|---|---|
| Release integrity | all hashes of `package.json` verified on the target (`deploy.verify`) | F2 |
| Tiles | `GET /v1/tiles/planet/{z}/{x}/{y}` 200/204, `tiles.json`, style, glyph, sprite; `/ready` | F4 |
| Tile reload | new `current.json` under load; zero 5xx (needs C2; today the container is recreated) | C2/F5 |
| Routing | cross-border route via routerservice | F4 |
| Geocode | street, interpolated address, transit stop | F4 |
| Regions | regionservice point/bbox/radius lookups, border crossings | F4 |
| Rollback | `flask deploy-rollback` (or the `/deploy` page) returns the previous release and its behaviour, per class | F5 |

Rollback is always: flip `current` to the previous release, repeat the class's activation action and remove the release that was rolled back (§3.4). Because releases are immutable and `previous` is retained, no rebuild is needed. The drill has not been run yet (F5).
