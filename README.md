# Sales Data Pipeline — Google Sheets → Supabase (n8n)

An automated ETL pipeline that consolidates sales data from multiple Google Sheets into a single Supabase (Postgres) warehouse table — deduplicated, normalized, and monitored by email in both directions (success + failure).

Built with **n8n** (low-code orchestration) and **Supabase** (Postgres warehouse).

## Why this exists

Manually exporting and reconciling sales data from multiple spreadsheets is slow and error-prone — different people maintain different sheets, formats drift, and duplicate/missing rows go unnoticed until someone builds a report on bad data.

This pipeline automates that: it runs daily, pulls from every connected sheet, normalizes them into one schema, removes duplicates, and lands clean data in a queryable Postgres table — with an email either confirming success (with row count) or flagging exactly what broke.

## Workflow diagram

Actual n8n workflow canvas:

![Sales Data Pipeline — n8n workflow](Sales%20Data%20Pipeline.png)

Simplified architecture view:

![Sales Data Pipeline architecture diagram](images/workflow-diagram.svg)

## Architecture

```
                 ┌─────────────────────┐
   06:00 daily → │  Schedule Trigger    │
                 └──────────┬───────────┘
                             │
         ┌───────────────────┼───────────────────┐
         ▼                   ▼                   ▼
   Fetch Sheet 1       Fetch Sheet 2       Fetch Sheet 3
         │                   │                   │
   Normalize            Normalize            Normalize
         │                   │                   │
         └───────────────────┼───────────────────┘
                             ▼
                    Merge (append all)
                             ▼
                 Remove Duplicates (by recordId)
                             ▼
                  Insert into Supabase (upsert-safe)
                             ▼
                     Count rows processed
                             ▼
                   Send success email ✅

   (any node failure) → Error Trigger → Send failure email ⚠️
```

## Stack

| Layer | Tool |
|---|---|
| Orchestration | [n8n](https://n8n.io) |
| Data sources | Google Sheets (any number of sheets) |
| Warehouse | [Supabase](https://supabase.com) (Postgres) |
| Alerts | Email (SMTP) |

## Repo contents

```
workflow/
  sales-data-pipeline.json   # n8n workflow export — import directly into n8n
supabase/
  schema.sql                 # Warehouse table schema
sample-data/
  sample_sales_data_1.csv    # Example sheets to test the pipeline with
  sample_sales_data_2.csv
  sample_sales_data_3.csv
```

## Setup

1. **Create the warehouse table** — run `supabase/schema.sql` in your Supabase project's SQL editor.
2. **Import the workflow** — in n8n: Workflows → Import from File → select `workflow/sales-data-pipeline.json`.
3. **Connect credentials:**
   - Google Sheets OAuth2 (on each "Fetch Google Sheet Orders" node)
   - Postgres, pointed at your Supabase project — **use the connection pooler host** (`aws-0-<region>.pooler.supabase.com`, port `6543`), not the direct `db.*.supabase.co` host, since many hosted n8n instances can't reach Supabase's direct IPv6-only endpoint.
   - SMTP (for success/failure emails)
4. **Point each "Fetch Google Sheet Orders" node** at a real spreadsheet + tab (the sample CSVs in `sample-data/` are a good way to test end-to-end first — import them into a Google Sheet each).
5. **Publish** the workflow in n8n (draft edits don't take effect until published).
6. Optional: run it manually once via n8n's "Execute workflow" button to confirm data lands in `sales_consolidated`.

## Extending it

- **More sheets:** duplicate a Fetch → Normalize branch, wire it into a new Merge input, bump `numberInputs` on the Merge node.
- **More sources (Shopify, Stripe, a CRM, etc.):** same pattern — fetch → normalize to the common schema (`source`, `recordId`, `recordDate`, `customer`, `amount`, `currency`) → merge in.
- **Faster sync:** swap the daily Schedule Trigger for an hourly one, or use n8n's Google Sheets Trigger node to poll for changes more frequently.

## Notes

- Deduplication is enforced both in-workflow (Remove Duplicates node) and at the database level (`UNIQUE` constraint on `recordId` + `ON CONFLICT DO NOTHING`), so re-running the workflow is always safe.
- Row Level Security is off by default on `sales_consolidated` since it's written to via a direct Postgres connection, not Supabase's public API. Enable it (see `schema.sql`) before exposing this project's API publicly.
