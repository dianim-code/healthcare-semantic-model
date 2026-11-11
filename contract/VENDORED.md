# Vendored gold contract

**Do not edit these files here.** They are a read-only copy of the gold data contract owned by
the `healthcare-lakehouse` repository (`docs/contract/` and `sample-data/gold/`). To change the
contract, change it there, bump `contract_version`, then re-vendor:

```bash
# from healthcare-lakehouse/
uv run scripts/vendor_contract.py --to <path-to-this-repo>
```

- **Contract version:** 1.1.0
- **Vendored on:** 2026-11-11
- **Human-readable contract:** `docs/data-contract.md` in `healthcare-lakehouse`

| File | SHA-256 (first 16) |
|---|---|
| `contract/gold_schema.yml` | `9c772c9030b3dd81` |
| `contract/reference/regions.csv` | `31ba7855c8933299` |
| `sample-data/gold/dim_discipline.csv` | `70d7435089f50bd6` |
| `sample-data/gold/dim_facility.csv` | `e4c57417110bcbc0` |
| `sample-data/gold/dim_region.csv` | `31ba7855c8933299` |
| `sample-data/gold/dim_year.csv` | `291bd9ce71927900` |
| `sample-data/gold/fact_hospital_beds.csv` | `978b8dae7a3e3600` |
| `sample-data/gold/fact_population.csv` | `4cd70ff7dd53af76` |

## Data attribution

The CSVs in `sample-data/gold/` are derived from data published by the **Ministero della Salute**
(Italian Ministry of Health) under the [Italian Open Data License v2.0](https://www.dati.gov.it/iodl/2.0/)
and by **ISTAT** under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The data has been
cleaned, re-keyed and aggregated; neither publisher endorses this project. Full source list:
`docs/data-sources.md` in `healthcare-lakehouse`.
