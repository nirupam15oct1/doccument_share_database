---
description: Build part of the Kite → Postgres (+optional Valkey) market-data pipeline from a task description, following the project's ingestion skills.
argument-hint: "[--valkey | --no-valkey] <what to build>"
---

## Task (from the user)

$ARGUMENTS

## How to build it

1. **Skills are authoritative.** Use `kite-live-ingest` for streaming/live work and
   `kite-historical-ingest` for candle backfills. Route every schema change and all SQL
   through `db-safety`. Never emit `DROP` / `DELETE` / `TRUNCATE`.

2. **Valkey toggle — read the flag in the task above:**
   - `--valkey` present → include the Valkey hot layer exactly as described in
     `kite-live-ingest` (write-through in the flush: `tick:<id>` latest-state hash,
     `ticks:<id>` Streams, `win:<id>` rolling-window sorted sets). Bring Valkey up with
     `docker compose --profile valkey up -d`, and connect via `VALKEY_URL`.
   - `--no-valkey`, or no flag → **do not** add Valkey. Serve live state from the Postgres
     `latest_tick` table instead. Bring up Postgres only with `docker compose up -d`.

3. **Universe is all Nifty 500 stocks.** Apply the "Data model at Nifty 500 scale"
   guidance: `instrument_id` surrogate key, day-partitioned `ticks`, month-partitioned
   `candles`, BRIN on `ts`, COPY-based batched writes.

4. **Config & secrets.** Postgres via `PG_DSN`; Kite `api_key` / `access_token` and DB
   creds from env / `.env` only — never hard-coded.

5. Lay out a 3–5 bullet plan first, then implement. Run nothing destructive.
