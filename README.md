# Healthcare Lakehouse and Decision Support Dashboard

Data project on synthetic healthcare records. Raw Synthea CSVs are ingested into a Databricks lakehouse with bronze, silver, and gold layers, modeled as a star schema, and surfaced through a Power BI dashboard built for a Decision Support Manager.

The data covers five U.S. states (California, Florida, New York, Pennsylvania, Texas) and roughly 20,000 patients across a 2021-2025 window.

## What's in this repo

```
healthcare-lakehouse-dashboard/
├── notebooks/              Databricks notebooks for each lakehouse layer
│   ├── 1_Bronze_Layer.ipynb      Raw ingestion + validation
│   ├── 2_Silver_Layer.ipynb      Cleaning, deduplication, scoping
│   └── 3_Gold_Layer.ipynb        Star schema with surrogate keys
├── dashboard/              Power BI report screenshots and link to .pbix
├── docs/                   Architecture, data model, validation notes
├── data/sample/            Small Synthea sample for inspection
└── README.md
```

## The stack

- **Synthea** for generating the source CSVs (16 tables, 5 state runs)
- **Databricks** with Unity Catalog volumes for storage
- **PySpark** for all transformations
- **Delta Lake** as the table format across all three layers
- **Power BI Desktop** for the dashboard

## Pipeline summary

### Bronze
Raw Synthea CSVs land in Unity Catalog volumes per state and get loaded into Delta tables asis. No cleaning, no deletions. A `source_state` column is added so downstream layers can scope by state. Validation runs after ingestion to check row counts, null patterns, and basic referential integrity. Anything unexpected is logged but not blocked.

### Silver
Bronze tables get cleaned: deduplicated on natural keys, scoped to the 20212025 analytical window, with text fields normalized (SNOMED axis suffixes stripped from descriptions, encounter classes standardized, race and ethnicity titlecased). Hashbased business keys are generated here. Deduplication is deterministic so reruns produce identical output.

### Gold
A star schema for analytics. Dimensions hold descriptive attributes (`dim_patient`, `dim_provider`, `dim_payer`, `dim_organization`, `dim_state`, `dim_date`). Facts hold measures and foreign keys (`fact_encounters`, `fact_claims`, `fact_clinical_events`, `fact_patient_utilization`). A `bridge_claim_diagnosis` table handles the many to many between claims and diagnosis codes. Integer surrogate keys are generated in gold so downstream consumers (Power BI, ad hoc queries, future ML work) join on integers rather than 32 character hashes.

The gold layer also documents three role playing payer dimensions (`dim_payer`, `dim_payer_primary`, `dim_payer_secondary`) for the Power BI model.

## Validation

Each layer runs its own validation suite and writes results to `healthcare_demo._metadata.validation_runs` and `validation_results`. Across all three layers there are 205 checks total. The audit trail makes it possible to ask "did the pipeline run clean last week?" in one SQL query.

See `docs/validation.md` for the full check list.

## The dashboard

A four page Power BI report built for the Decision Support Manager persona (reporting to the COO). The pages cover:

1. **Executive Summary:** KPI cards with year over year deltas, encounter trend, payer mix, outstanding balance breakdown
2. **Encounter Activity:** volume trends by class, encounters per patient, top encounter reasons, provider specialty mix, duration distribution
3. **Financial / Claims:** claim volume, outstanding balances by responsible party (primary / secondary / patient), payer concentration
4. **Geographic / Demographic:** patient population profile, growth by state, demographic shifts

**Download the .pbix**: [Leadership_Dashboard.pbix on Google Drive](https://drive.google.com/file/d/17neA8vSktP5mFrtYOyvoEtV5xuenY1R6/view?usp=sharing) (43 MB). Hosted on Drive rather than in the repo because it's a binary file that doesn't diff well in git. Screenshots of each page are in `dashboard/screenshots/`.
## Running it yourself

If you want to reproduce the pipeline:

1. Generate Synthea data for the five states with a 2021-2025 window. The exact command is in `docs/architecture.md`.
2. Upload the CSVs into Unity Catalog volumes under `healthcare_demo` (or change the catalog name at the top of the bronze notebook).
3. Run the notebooks in order: bronze, silver, gold.
4. Open the .pbix and point the connection to your Databricks SQL warehouse.

## About this project

This was built as a portfolio piece to work through a complete lakehouse to dashboard project. The data is synthetic (Synthea), so any numbers shown should be read as illustrative rather than clinical. The architecture, validation patterns, and modeling choices are the parts I'd defend in a code review.

Comments and suggestions are welcome.
