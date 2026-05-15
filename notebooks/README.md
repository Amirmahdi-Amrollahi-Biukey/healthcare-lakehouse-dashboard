# Notebooks

The three notebooks in this folder make up the full pipeline. They are intended to be run in order on Databricks against Unity Catalog.

| Notebook | What it does |
|----------|--------------|
| `1_Bronze_Layer.ipynb` | Loads Synthea CSVs from Unity Catalog volumes into bronze Delta tables. Adds a `source_state` column. Runs bronze validation. |
| `2_Silver_Layer.ipynb` | Reads bronze, deduplicates, scopes to 2021-2025, normalizes text fields, generates hash business keys. Runs silver validation. |
| `3_Gold_Layer.ipynb` | Builds the star schema (dimensions, facts, bridge). Generates integer surrogate keys. Runs gold validation. |

Drop your finalized Bronze / Silver / Gold notebooks (the cleaned, GitHub-ready versions from the previous session) into this folder before pushing the repo.

## Catalog and schema names used

- Catalog: `healthcare_demo`
- Bronze schema: `healthcare_bronze`
- Silver schema: `healthcare_silver`
- Gold schema: `healthcare_gold`
- Metadata schema (validation history): `_metadata`

If you want to use different names, change the `catalog_name`, `bronze_schema`, `silver_schema`, `gold_schema` variables defined near the top of each notebook.

## Source volumes

The bronze notebook expects Synthea CSVs under:

```
/Volumes/healthcare_demo/healthcare_bronze/california/
/Volumes/healthcare_demo/healthcare_bronze/florida/
/Volumes/healthcare_demo/healthcare_bronze/newyork/
/Volumes/healthcare_demo/healthcare_bronze/pennsylvania/
/Volumes/healthcare_demo/healthcare_bronze/texas/
```

Each folder should contain the standard Synthea CSV exports (`patients.csv`, `encounters.csv`, etc.).
