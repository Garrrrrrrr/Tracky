---
name: "Project Plan: Multi-Source Stock Tracker (Web)"
overview: A step-by-step build plan to modernize your Stock Tracker into a web app that ingests from multiple sources (scraping + APIs), stores to Supabase/Postgres, and combines via derived joins.
todos:
  - id: scaffold-nextjs-prisma
    content: Scaffold Next.js + TypeScript, add Prisma, connect to Supabase locally, and set up env var handling.
    status: pending
  - id: schema-canonical-enrichment
    content: "Create Supabase/Postgres schema: canonical trades tables and enrichment tables based on `DATA`/`SPECDATA` columns in `WebscrapeandDB.py`."
    status: pending
  - id: derived-view
    content: Create the derived combined dataset as a SQL VIEW (or materialized view later) that joins canonical trades with enrichment fields.
    status: pending
  - id: openinsider-scraper-ts
    content: Port the OpenInsider scraping/parsing logic from `WebscrapeandDB.py` into a TypeScript ingestion module that returns normalized canonical records.
    status: pending
  - id: external-api-ingestor
    content: Implement an ingestion module for your external API source(s) that maps API payloads into the same canonical record shape.
    status: pending
  - id: ingest-endpoints
    content: Add Next.js API routes to trigger each ingestion module and upsert results with idempotency + ingestion stats.
    status: pending
  - id: scheduler-cron
    content: Add Vercel Cron (or equivalent) to periodically run general ingestion and record run logs.
    status: pending
  - id: ui-trades-and-export
    content: Implement trade browsing UI (table with sorting/filtering/pagination) and export endpoints (CSV/JSON).
    status: pending
  - id: combine-ui
    content: Update UI to read from the derived combined view and display merged/enriched fields.
    status: pending
  - id: tests-fixtures-upsert
    content: "Add tests: OpenInsider HTML parsing fixtures, and idempotency/integration tests for upsert and join logic."
    status: pending
isProject: false
---

# Project Plan: Multi-Source Stock Tracker (Web)

## Assumptions (based on your answers)

- Target platform: web
- DB/hosting: Supabase (Postgres)
- Auth: not needed right away (build without auth; add later)
- Data combining mode: keep sources separate, then create a derived “combined dataset” using joins (`join_derived`)
- Updates: periodic sync (scheduled jobs)

## Target Tech Stack (recommended default)

- Frontend/UI: Next.js (React) + TypeScript
- DB: Supabase Postgres
- ORM/migrations: Prisma
- Data fetching: TanStack Query
- Table UI: TanStack Table
- Scraping/ingestion: TypeScript ingestion modules (server-side) and/or a small Python worker later if needed
- Scheduling: Vercel Cron (or Supabase scheduled functions / external cron later)

## Map from your current project to the new one

Your existing app has:

- Scraping logic in `WebscrapeandDB.py` (OpenInsider HTML parsing + SQLite schema + upserts)
- UI orchestration in `main.py` (build OpenInsider URLs + trigger scrape + export)

Key migration anchors:

- Your DB schema columns in `WebscrapeandDB.py` (`DATA` and `SPECDATA`) become Postgres tables (or normalized/canonical tables) in Supabase.
- Your URL-building/query parameters in `main.py` become part of the ingestion “job definitions.”

## High-level architecture

```mermaid
flowchart LR
  UI[Web UI
  tables/filters/export] -->|API calls| API[Next.js API routes]

  API --> IngestScrape[IngestScrape
  (OpenInsider scraper)]
  API --> IngestAPI[IngestAPI
  (external API clients)]

  IngestScrape --> Canonical[Canonical tables
  normalized trades]
  IngestAPI --> Enrich[Enrichment tables
  source-specific fields]

  Canonical --> Combined[Derived combined dataset
  SQL VIEW/MV with joins]
  Enrich --> Combined

  Combined -->|query results| UI
```



## Data model approach for `join_derived`

Because you want to combine multiple sources without losing provenance, use a layered model:

### 1) Per-source tables (optional but strongly recommended)

Store raw or minimally processed source results so you can reprocess when parsing/API changes.

- `raw_sources` (or `raw_<source>`) with `source_name`, `source_record_id`, `payload_jsonb`, `ingested_at`

### 2) Canonical tables (normalized)

Create canonical tables that represent the shared concept you browse in the UI (e.g., “trade events”).

- Example: `trades_canonical` with columns similar to `DATA` + `SPECDATA` from your current project.

### 3) Enrichment tables

Create tables for fields that come from only some sources (e.g., an external API).

- Example: `trades_api_enrichment` keyed by the canonical trade key.

### 4) Derived combined dataset

Create SQL views/materialized views that join canonical + enrichment:

- `v_trades_combined` (VIEW) for simplicity
- later upgrade to a materialized view if performance becomes an issue

### Dedup/upsert strategy

- Define a stable unique key for each canonical trade record.
- Use `INSERT ... ON CONFLICT DO UPDATE` (via Prisma upserts) so periodic sync is safe and idempotent.

## Build phases (in order)

### Phase 0: Project scaffolding

1. Scaffold Next.js + TypeScript.
2. Add Prisma + Supabase connection.
3. Add environment variable setup:
  - `DATABASE_URL` (Supabase)
  - `OPENINSIDER_BASE_URL` (optional)
  - API keys for any external API sources (kept server-side)

### Phase 1: Define the database schema

1. Translate your SQLite tables into Postgres tables.
  - Mirror columns from `DATA` and `SPECDATA` defined in `WebscrapeandDB.py`.
2. Add canonical and enrichment layering.
3. Create SQL views for combined data (initially as a simple VIEW).

Note: during early development, you can keep it simple by using one canonical table for both “general” and “specific” trades, with a `dataset_type` column.

### Phase 2: Implement ingestion modules

Build two independent ingestion modules first:

1. OpenInsider scraper module (port from `WebscrapeandDB.py`)
  - `getGeneralTrade` equivalent
  - `getSpecificTrade` equivalent
2. External API client module
  - normalizes external records into the same canonical shape

Both should output normalized records of the same “canonical trade” type.

### Phase 3: Ingestion endpoints (API routes)

Expose endpoints like:

- `POST /api/ingest/openinsider/general`
- `POST /api/ingest/openinsider/specific`
- `POST /api/ingest/<external-source>`

Each endpoint:

- fetches from its source
- validates/transforms into canonical format
- upserts into Supabase
- returns ingestion stats (counts, inserted/updated)

### Phase 4: Periodic scheduling

1. Add Vercel Cron (or equivalent) to call the ingestion endpoints.
2. First schedule: general trade ingestion daily (or your preferred interval).
3. Later schedule: specific ingestion patterns (ticker/day filters) if you have a reason.

### Phase 5: UI (browse + filter + export)

1. Trades pages:
  - `/trades/general` and `/trades/specific` (optional) or a single `/trades` with `dataset_type` filter
2. Table features:
  - sort (date, ticker, trade type)
  - filter/search (ticker, trader name, etc.)
  - pagination
3. Export:
  - `GET /api/export/trades.csv?filters...`
  - `GET /api/export/trades.json?filters...`

This mirrors your original CSV export behavior from `main.py` / `pandas.to_csv`.

### Phase 6: Derived combined view + join logic

Once both ingestion sources exist:

- implement `v_trades_combined` that joins canonical trades with API enrichment
- update UI to read from the combined view

### Phase 7: “Analysis/ML-ready” scaffolding

Even if you’re not sure yet, prepare a place to write analysis results:

- `trade_features` and/or `trade_predictions` tables
- optional background job later

## Testing strategy (recommended for this kind of project)

- Unit tests for HTML parsing:
  - store a few OpenInsider HTML fixtures locally
  - verify the scraper extracts consistent fields
- Integration tests for upsert:
  - run ingestion twice and ensure row counts are stable (idempotency)
- Contract tests for the external API normalization:
  - validate schema mapping and types

## Notes on “anything else of note”

- CORS: UI never talks directly to external APIs; ingestion endpoints run server-side.
- Rate limits: add simple backoff/retry in ingestion modules.
- Provenance: add `source_name` and `ingestion_run_id` so you can trace rows back to the run.

## Key files you’ll port/translate from the current project

- `[WebscrapeandDB.py](WebscrapeandDB.py)`: scraping + SQLite schema (translate tables and parsing logic)
- `[main.py](main.py)`: URL construction + ingestion triggers + exports (translate into ingestion jobs + API endpoints)

