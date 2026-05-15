# Architecture

## Overview

Three-layer lakehouse on Databricks, source data from Synthea, output consumed by Power BI.

```
Synthea CSVs (5 states)
        │
        ▼
  Unity Catalog volumes (one folder per state)
        │
        ▼
   ┌─────────┐
   │ Bronze  │  Raw ingestion, no transforms. Validation logged.
   └────┬────┘
        ▼
   ┌─────────┐
   │ Silver  │  Cleaning, dedup, scoping, normalization.
   └────┬────┘
        ▼
   ┌─────────┐
   │  Gold   │  Star schema. Surrogate keys. Power BI reads from here.
   └────┬────┘
        ▼
  Power BI Desktop (Import mode via Databricks SQL warehouse)
```

All three layers store Delta tables. The catalog is `healthcare_demo` with schemas `healthcare_bronze`, `healthcare_silver`, `healthcare_gold`, and `_metadata` for validation history.

## Source data

Synthea is a synthetic patient generator. It produces CSV files that mirror the structure of real EHR exports without using any real patient data.

Generation command used for this project:

```bash
java -jar synthea-with-dependencies.jar \
  -s 12345 \
  -p 6000 \
  --exporter.years_of_history 5 \
  --exporter.fhir.export false \
  --exporter.csv.export true \
  California
```

Repeat for each state with different seeds. Output goes to `output/csv/` and gets uploaded to Unity Catalog volumes.

Sixteen tables come out of Synthea: `patients`, `encounters`, `claims`, `claims_transactions`, `conditions`, `medications`, `procedures`, `observations`, `allergies`, `immunizations`, `careplans`, `imaging_studies`, `devices`, `payers`, `payer_transitions`, `providers`, `organizations`, `supplies`.

## Layer responsibilities

### Bronze: raw faithful copy

Bronze tables match the source CSV schema exactly, with one addition: a `source_state` column so we know which state run produced each row. Nothing else changes.

This is the layer to fall back on if anything downstream looks wrong. If silver produced bad numbers, you can re-derive silver from bronze without re-running Synthea.

Bronze validation logs the counts and basic schema observations but does not block the pipeline. Synthea sometimes emits expected anomalies (e.g., a small number of organization-level duplicates across state runs); these are documented warnings rather than errors.

### Silver: clean and conformed

Silver is where data quality work happens.

- **Deduplication** on natural keys with deterministic tie-breaking. Reruns produce identical output.
- **Scoping** to the 2021-2025 analytical window. Encounter-linked event tables (conditions, procedures, etc.) are scoped to encounters within the window, so the data stays internally consistent.
- **Normalization** of text fields. SNOMED axis labels like `(disorder)`, `(procedure)`, `(finding)` get stripped from description columns using a whitelist regex. Encounter classes get standardized casing (so `Snf` becomes `SNF`, etc.). Race and ethnicity fields are title-cased.
- **Hash business keys** are generated per row to anchor each silver record to a stable identifier.

A few quality decisions worth flagging:

- The original `description` columns are preserved alongside `description_clean`, so anyone reading the silver tables can trace a cleaned value back to its raw form.
- State-level scoping is per table type: longitudinal tables (patients, providers, organizations) keep all rows; event tables (encounters, claims, etc.) get filtered to the analytical window.

### Gold: dimensional model

Gold is the layer Power BI and other consumers read from. The shape is a star schema in the Kimball style.

**Dimensions:**
- `dim_patient`
- `dim_provider`
- `dim_organization`
- `dim_payer` (plus two role-playing instances for primary/secondary in Power BI)
- `dim_state`
- `dim_date`

**Facts:**
- `fact_encounters`
- `fact_claims`
- `fact_clinical_events` (conditions, procedures, observations, medications unified for clinical analysis)
- `fact_patient_utilization` (one row per patient with summary metrics)

**Bridge:**
- `bridge_claim_diagnosis` (handles the many-to-many between claims and diagnosis codes, with diagnosis position preserved)

**Surrogate keys:**

Each dimension gets an integer surrogate key generated with `row_number()` over a window on the silver hash key. Facts carry the integer surrogate keys instead of (or alongside) the original string IDs. This is mainly a performance choice: integer joins are cheaper in Spark and they compress much better in Power BI's columnar engine. The hash-based business keys from silver are preserved on dimensions for traceability.

**Column ordering** in gold tables follows the standard convention: surrogate key, then foreign keys, then degenerate dimensions, then descriptive attributes, then measures. This makes the schema readable at a glance when someone opens the table for the first time.

## Why three layers

The split exists because each layer is solving a different problem:

- Bronze answers "what did the source actually give us?"
- Silver answers "what does the cleaned, conformed version of that look like?"
- Gold answers "what shape do analysts need for the questions they're asking?"

Putting all of this in one notebook would conflate concerns. Putting cleaning in gold would make bronze useless. Putting modeling in silver would make silver expensive to rebuild. The three-layer split lets each layer be cheap to rerun on its own when something upstream changes.

## Refresh

The Power BI report uses Import mode against the gold tables via a Databricks SQL warehouse. Refresh is manual in Power BI Desktop for this project. In a production setup, the same architecture would publish to Power BI Service and schedule refresh after the gold notebook completes.
