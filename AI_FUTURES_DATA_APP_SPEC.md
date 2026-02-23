---
title: "Futures Data Pipeline App Spec"
version: "1.0"
owner: "User"
target_agent: "General coding AI agent"
language: "Python 3.11+"
framework: "Streamlit"
database: "SQLite"
timeseries_store: "Parquet"
app_profile: "futures"
root_path: "/Users/user/Python-Projects/RST"
provider: "Databento"
provider_defaults:
  dataset: "GLBX.MDP3"
  schemas:
    - "ohlcv-1d"
    - "definition"
  stype_in: "parent"
  stype_out: "instrument_id"
constraints:
  - "Futures-only scope."
  - "Daily-frequency data only."
  - "Raw files are immutable once downloaded."
  - "No destructive DB reset unless explicitly requested."
validation_commands:
  - "python -m py_compile streamlit_app.py src/rst/data/*.py"
  - "streamlit run streamlit_app.py"
---

# Goal
Build a local web app that manages futures data end-to-end for systematic trend-following research:
- ingest raw contract-level data,
- update on demand via API,
- maintain curated daily datasets,
- build continuous futures series from configurable roll definitions,
- inspect/validate data in UI.

# Scope / Non-Scope

## In Scope
- Futures only
- Daily OHLCV data
- Definitions ingestion (metadata by contract)
- Continuous futures construction
- Metadata management (SQLite)
- Streamlit views for update + inspection

## Out of Scope
- Strategy backtesting
- Signal generation
- Broker execution/trading

# Requirements

## Functional
1. Ingest local raw files into curated contract-level parquet.
2. Download + ingest new data from Databento on demand.
3. Ingest local definition files separately.
4. Store symbol mappings, definitions, coverage, and parent metadata in SQLite.
5. Build full and incremental continuous series from saved rolling definitions.
6. Visualize contract and continuous data in Streamlit.

## Databento-Specific
1. Use Databento Historical API as the default provider.
2. Default dataset: `GLBX.MDP3`.
3. Required schemas:
   - `ohlcv-1d` for daily contract bars.
   - `definition` for instrument metadata.
4. API key loaded from `config/databento.env` as `DATABENTO_API_KEY`.
5. Request ranges based on existing coverage and download only missing windows.
6. Handle Databento 422 range/entitlement errors explicitly with actionable UI messages.

## Non-Functional
1. Reproducible and idempotent ingestion behavior.
2. Raw data never mutated.
3. Explicit error messages in UI for missing files/schema/invalid date ranges.
4. Graceful handling of missing values.
5. Widget keys unique across entire Streamlit app.

# Data Model / Schema

## `ingest_files`
Tracks ingested raw files to avoid duplicate processing.
- `path` (PK)
- `dataset`
- `schema`
- `start_date`
- `end_date`
- `ingested_at`

## `symbol_coverage`
Coverage window per symbol and schema.
- `symbol` (PK part)
- `dataset` (PK part)
- `schema` (PK part)
- `start_date`
- `end_date`
- `last_updated`

## `instrument_map`
Parent-to-contract mapping.
- `parent` (PK part)
- `symbol` (PK part)
- `instrument_id`
- `dataset` (PK part)
- `schema` (PK part)
- `first_date`
- `last_date`
- `source_path`

## `instrument_definitions`
Contract definitions over time.
- `parent`
- `symbol`
- `instrument_id`
- `activation`
- `expiration`
- `exchange`
- `currency`
- `tick_size`
- `contract_multiplier`
- `unit_of_measure_qty`
- `tick_value`
- `instrument_class`
- `security_type`
- `raw_json`
- `dataset`, `schema`

## `parent_metadata`
Parent-level fields used for roll/build logic.
- `parent` (PK)
- `exchange`
- `timezone`
- `currency`
- `tick_size`
- `contract_multiplier`
- `contract_notional` (optional override)
- `roll_days`
- `adjustment_method` (`ratio|additive|none`)
- `instrument_class`
- `security_type`
- `notes`

## `continuous_series_definitions`
Saved rules to build/update a continuous series.
- `series_id` (PK)
- `name`
- `parent`
- `rolling_criteria` (initially: `days_before_expiry`)
- `roll_days`
- `adjustment_method`
- `price_field`
- `contract_rank`
- `include_months`
- `use_spreads`
- `instrument_class`
- `security_type`
- `created_at`
- `updated_at`
- `notes`

## `continuous_series_runs`
Audit log for build/update runs.
- `run_id` (PK)
- `series_id`
- `start_date`
- `end_date`
- `built_at`
- `output_path`
- `rows`
- `source_info`

# File Structure
```text
project/
  streamlit_app.py
  src/
    rst/
      data/
        metadata.py
        ingest.py
        definitions.py
        provider_api.py
      continuous/
        builder.py
  config/
    databento.env         # gitignored, contains DATABENTO_API_KEY
  data/
    metadata.db
    raw/
      ohlcv-1d/
        <PARENT>/
      definition/
        <PARENT>/
    curated/
      ohlcv-1d/
      continuous/
```

# Workflows

## 1) Ingest Local Raw Files
1. User chooses `raw_base_dir`, `curated_dir`, `db_path`.
2. App lists raw folders containing data files.
3. User selects parents/folders and runs ingestion.
4. Parser writes per-contract parquet to curated folder.
5. App updates `instrument_map`, `symbol_coverage`, and `ingest_files`.

## 2) Download + Ingest (On Demand)
1. User sets Databento config, dataset, schema.
2. App computes planned date ranges per parent from coverage table.
3. Optional Databento cost estimate.
4. Download raw files to `data/raw/ohlcv-1d/<parent>/`.
5. Ingest downloaded files into curated + DB updates.

## 3) Ingest Local Definition Files
1. User selects definitions base directory and parents.
2. App parses local definition files and upserts `instrument_definitions`.
3. Optional auto-fill missing parent metadata from definitions.

## 4) Continuous Series Build
1. User creates/saves rolling definition.
2. App infers valid contracts from mappings/definitions and month filters.
3. Full build or incremental update (with lookback window).
4. Write output parquet and run log.

# Acceptance Criteria
1. App ingests contract-level data for at least 4 futures parents.
2. Definitions viewer shows contracts ordered by expiry with key fields.
3. Parent metadata editor saves/reloads correctly.
4. Continuous series builds without crashes and writes parquet outputs.
5. Incremental update does not duplicate history.
6. App loads cleanly (no syntax/runtime/widget-id errors).

# Constraints
1. Futures-only feature set in this app.
2. Daily bars only.
3. Databento API key loaded from env file (`config/databento.env`) and never committed.
4. Raw files are append-only archive; no overwrite/mutation.
5. No destructive command or schema drop without explicit user request.

# Step-by-Step Implementation Order
1. Implement DB schema + connection/migrations (`metadata.py`).
2. Implement local raw ingestion pipeline (`ingest.py`).
3. Implement definitions ingestion parser (`definitions.py`).
4. Implement provider API wrapper (`provider_api.py`).
5. Implement continuous builder (`continuous/builder.py`).
6. Build Streamlit `Data Updates` tab with 3 modes:
   - local ingest,
   - download+ingest,
   - local definitions ingest.
7. Build `Symbol Browser`, `Parent Metadata`, `Definitions Viewer`.
8. Build `Continuous Series` tabs (definitions/create-update/browser).
9. Add validations and run acceptance checks.
10. Final pass: README + operator instructions.

# Do Not
1. Do not hide ingestion errors; surface actionable messages.
2. Do not silently coerce broken date ranges.
3. Do not rewrite raw source files.
4. Do not reset DB/data directories unless explicitly instructed.
