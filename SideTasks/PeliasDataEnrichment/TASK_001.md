# TASK_001 — Fix OpenAddresses Coverage Gaps

**Status**: Done

## Overview

Audit and fix missing OpenAddresses files in `config/config-mac.yml`. This is a config-only change — no pipeline code modifications required.

All gaps below have been validated against the live OA GitHub source repository
(`https://github.com/openaddresses/openaddresses/tree/master/sources`).
Each gap is marked ✅ (data exists, add it), ⚠️ (partial data only), or ❌ (not in OA).

## File to Change

- `data-pipeline/config/config-mac.yml`

---

## Gap 1 — Germany: 4 missing Bundesländer

Every region that includes Germany has 12 of 16 Bundesländer. Status per missing state:

| OA file | State | Status | Notes |
|---|---|---|---|
| `de/ni/statewide.csv` | Lower Saxony | ✅ Available | `sources/de/ni/statewide.json` exists |
| `de/mv/statewide.csv` | Mecklenburg-Vorpommern | ✅ Available | `sources/de/mv/statewide.json` exists |
| `de/berlin.csv` | Berlin | ✅ Available | Source is `sources/de/berlin.json` — file name is **`berlin.csv`**, not `de/be/statewide.csv` |
| `de/by/statewide.csv` | Bavaria | ❌ Not available | No statewide file — OA only has 7 sub-regional files (see below) |

**Bavaria sub-regional files (✅ all exist):**
```
de/by/region_lower_bavaria.csv
de/by/region_lower_franconia.csv
de/by/region_middle_franconia.csv
de/by/region_swabia.csv
de/by/region_upper_bavaria.csv
de/by/region_upper_franconia.csv
de/by/region_upper_palatinate.csv
```

### What to add

Add to every region that already lists `de/` Bundesländer:

```yaml
# In each affected region's openaddresses section, alongside existing de/ entries:
- de/ni/statewide.csv
- de/mv/statewide.csv
- de/berlin.csv
- de/by/region_lower_bavaria.csv
- de/by/region_lower_franconia.csv
- de/by/region_middle_franconia.csv
- de/by/region_swabia.csv
- de/by/region_upper_bavaria.csv
- de/by/region_upper_franconia.csv
- de/by/region_upper_palatinate.csv
```

**Affected regions:**

| Region | Section |
|---|---|
| `baltics-russia` | overlap |
| `central-europe` | overlap |
| `scandinavia` | overlap |
| `south-europe` | overlap |
| `west-europe` | core |

---

## Gap 2 — Sweden: 2 municipalities missing from config

`se/countrywide.csv` ❌ does not exist in OA — per-municipality files are the only option.

OA has 16 municipality files. The config references 14 of them. Two are missing:

| OA file | Status |
|---|---|
| `se/municipality_of_kristinehamn.csv` | ✅ Available — missing from all regions |
| `se/municipality_of_Österåker.csv` | ✅ Available — missing from all regions |

### What to add

Add to every region that already lists Sweden municipalities:

```yaml
- se/municipality_of_kristinehamn.csv
- se/municipality_of_Österåker.csv
```

**Affected regions:**

| Region | Section |
|---|---|
| `baltics-russia` | overlap |
| `central-europe` | overlap |
| `scandinavia` | core |
| `west-europe` | overlap |

---

## Gap 3 — `west-europe` core: Netherlands and Belgium Wallonia French

| OA file | Status | Notes |
|---|---|---|
| `nl/countrywide.csv` | ✅ Available | `sources/nl/countrywide.json` exists. NL is a core country in `west-europe` but has no OA entry in core (it is correctly in `uk-iceland` overlap) |
| `be/wal/bosa-region-wallonia-fr.csv` | ✅ Available | `sources/be/wal/bosa-region-wallonia-fr.json` exists. The `-de` variant is already in `west-europe` core; `-fr` is missing |

### What to add

```yaml
# west-europe > core > openaddresses — add:
- nl/countrywide.csv
- be/wal/bosa-region-wallonia-fr.csv
```

---

## Gap 4 — `uk-iceland` core: No UK, Ireland, or Isle of Man data

| OA file | Status | Notes |
|---|---|---|
| `gb/...` | ❌ Not in OA | No `sources/gb/` directory exists |
| `ie/countrywide.csv` | ❌ Not in OA | No `sources/ie/` directory exists |
| `im/countrywide.csv` | ❌ Not in OA | No `sources/im/` directory exists |

**No action possible** — these are not available in OpenAddresses.

---

## Gap 5 — `central-europe` core: Hungary, Romania, Moldova, Ukraine

| OA file | Status | Notes |
|---|---|---|
| `hu/countrywide.csv` | ❌ Not in OA | No `sources/hu/` directory exists |
| `ro/countrywide.csv` | ❌ Not in OA | Only Bucharest data: `ro/bucharest-metro.csv` and `ro/bucharest.csv` |
| `md/countrywide.csv` | ❌ Not in OA | No `sources/md/` directory exists |
| `ua/...` | ⚠️ Partial | Only 2 cities: `ua/12/city_of_dnipropetrovsk.csv` and `ua/63/city_of_kharkiv.csv` |

**No useful action** — coverage is too sparse to be worth adding for a routing app.

---

## Gap 6 — `south-europe` core: Bosnia and Malta

| OA file | Status | Notes |
|---|---|---|
| `ba/countrywide.csv` | ❌ Not in OA | No `sources/ba/` directory exists |
| `mt/countrywide.csv` | ❌ Not in OA | No `sources/mt/` directory exists |

**No action possible.**

---

## Gap 7 — `south-eastern-europe` core: sparse coverage

| OA file | Status | Notes |
|---|---|---|
| `al/countrywide.csv` | ❌ Not in OA | No `sources/al/` directory exists |
| `gr/countrywide.csv` | ❌ Not in OA | Only 1 file: `gr/b/municipality_of_kalamaria.csv` — not worth adding |
| `me/countrywide.csv` | ❌ Not in OA | No `sources/me/` directory exists |
| `mk/countrywide.csv` | ❌ Not in OA | No `sources/mk/` directory exists |
| `tr/...` | ❌ Not in OA | No `sources/tr/` directory exists |

**No action possible.**

---

## Gap 8 — `iberian-peninsula` overlap: UK and Ireland

| OA file | Status |
|---|---|
| `ie/countrywide.csv` | ❌ Not in OA |
| `gb/...` | ❌ Not in OA |

**No action possible.**

---

## Alternatives for Countries Not in OpenAddresses

Validated via web search March 2026. Each country below has no OA source and was researched for alternatives.

| Country | Best free alternative | Quality | Notes |
|---|---|---|---|
| Ireland (ie) | None | — | Eircode (complete, geocoded) is **commercial only**. OSi Open Data has boundaries but no address-level data. GeoDirectory is paid. OSM only. |
| Isle of Man (im) | None | — | No open address dataset found. Small territory; OSM coverage is reasonable. |
| UK/GB (gb) | OS AddressBase Open | ★★★★★ | See **TASK_005**. Free with registration on OS Data Hub. Great Britain only; Northern Ireland has no free alternative. |
| Hungary (hu) | Overture `addresses` | ★★★☆☆ | National land registry is paid. No free national address registry found. Overture covers HU. See **TASK_003**. |
| Romania (ro) | Overture `addresses` | ★★★☆☆ | ANCPI geoportal requires free registration but focuses on cadastral parcels, not housenumber addresses. Only Bucharest in OA. Overture covers RO. See **TASK_003**. |
| Moldova (md) | None practical | ★☆☆☆☆ | date.gov.md has 1,274 open datasets but no address registry found. Very low priority — small country; OSM covers major towns. |
| Ukraine (ua) | data.gov.ua (investigate) | ★★☆☆☆ | Ukraine scores 97% on EU Open Data Maturity (3rd in Europe). data.gov.ua has 80,000+ datasets — an address registry may exist but requires manual investigation of the portal. Otherwise Overture. |
| Bosnia-Herzegovina (ba) | None | — | No open address data found. No national open data portal with address data. OSM only. |
| Malta (mt) | Malta GeoHub (investigate) | ★★☆☆☆ | Malta GeoHub (geohub.gov.mt) is the national spatial data portal. open.data.gov.mt exists with datasets. Could not confirm address-level data — requires manual check of the portal. |
| Albania (al) | None | — | No open address data found. OSM only. |
| Greece (gr) | Overture `addresses` | ★★☆☆☆ | geodata.gov.gr has CSV datasets but only administrative/boundary data confirmed. Commercial geodata.gr has an address+postcode DB (paid). Overture covers GR. See **TASK_003**. |
| Montenegro (me) | None | — | No open address data found. OSM only. |
| North Macedonia (mk) | None | — | No open address data found. OSM only. |
| Turkey (tr) | None (MAKS not open) | — | MAKS (Mekansal Adres Kayıt Sistemi) is Turkey's official national address registry — comprehensive but **not publicly downloadable**. Internal government system only. OSM only. |

### Items requiring manual investigation before writing off

- **Ukraine** — Visit [data.gov.ua](https://data.gov.ua/en/dataset?res_format=CSV) and search for "address" or "адреса". Ukraine has a strong open data culture and a registry may exist.
- **Malta** — Visit [open.data.gov.mt](https://open.data.gov.mt/datasets.html) and [geohub.gov.mt](https://geohub.gov.mt) and check for address/property datasets.

---

## Summary: What to Actually Do

| Action | Regions affected |
|---|---|
| Add `de/ni/statewide.csv` | baltics-russia (overlap), central-europe (overlap), scandinavia (overlap), south-europe (overlap), west-europe (core) |
| Add `de/mv/statewide.csv` | same 5 regions |
| Add `de/berlin.csv` | same 5 regions |
| Add 7× `de/by/region_*.csv` files | same 5 regions |
| Add `se/municipality_of_kristinehamn.csv` | baltics-russia (overlap), central-europe (overlap), scandinavia (core), west-europe (overlap) |
| Add `se/municipality_of_Österåker.csv` | same 4 regions |
| Add `nl/countrywide.csv` | west-europe (core) |
| Add `be/wal/bosa-region-wallonia-fr.csv` | west-europe (core) |

---

## Acceptance Criteria

- [ ] `de/ni/statewide.csv` added to all 5 affected regions
- [ ] `de/mv/statewide.csv` added to all 5 affected regions
- [ ] `de/berlin.csv` added to all 5 affected regions (note: NOT `de/be/statewide.csv`)
- [ ] All 7 `de/by/region_*.csv` files added to all 5 affected regions
- [ ] `se/municipality_of_kristinehamn.csv` added to all 4 Sweden regions
- [ ] `se/municipality_of_Österåker.csv` added to all 4 Sweden regions
- [ ] `nl/countrywide.csv` added to `west-europe` core
- [ ] `be/wal/bosa-region-wallonia-fr.csv` added to `west-europe` core

## Testing Notes

After config changes, run a test pipeline build on one affected region. Check that the new files download and import without error. Confirm new `address` records appear in Elasticsearch for the added areas (e.g. a Munich address for the Bavaria regional files, a Rostock address for `de/mv`).
