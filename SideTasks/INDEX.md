# Index - Side Tasks

| Task | Title | Status |
| --- | --- | --- |
| [PELIAS_DATA_ENRICHMENT](./PELIAS_DATA_ENRICHMENT.md) | Pelias Data Enrichment Plan | Living doc — see tasks below |
| [PeliasDataEnrichment/TASK_001](./PeliasDataEnrichment/TASK_001.md) | Fix OpenAddresses Coverage Gaps | Done |
| [PeliasDataEnrichment/TASK_002](./PeliasDataEnrichment/TASK_002.md) | Add Pelias Polylines Importer | Done |
| [PeliasDataEnrichment/TASK_003](./PeliasDataEnrichment/TASK_003.md) | Add Overture Maps Places & Addresses | Done |
| [PeliasDataEnrichment/TASK_004](./PeliasDataEnrichment/TASK_004.md) | Add GTFS Transit Stops | Done (importer) — feeds disabled pending source verification |
| [PeliasDataEnrichment/TASK_005](./PeliasDataEnrichment/TASK_005.md) | Add UK Address Data (OS AddressBase Open) | Planned |

> TASK_001–TASK_004 were implemented in the legacy, **deprecated** `data-pipeline`. The equivalent now lives in `data-manager`'s Pelias stages: `datamanager/stages/pelias.py` (importers), `datamanager/services/pelias_data.py` (importer configuration), `datamanager/services/pelias_sources.py` and `datamanager/services/overture.py` (source downloads), `datamanager/stages/pelias_interpolation.py` and `datamanager/services/pelias_interpolation.py` (interpolation databases), `download-overture-gtfs`. The paths to `data-pipeline/config/*.yml` and `pipeline/pelias_funcs.py` in the task files are kept as history; TASK_005 is still planned. See [MIGRATION-DATA-MANAGER](../MIGRATION-DATA-MANAGER.md).
