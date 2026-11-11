# CLAUDE.md — Kite → Postgres Live Streaming market-data pipeline (orchestrator context)

You are the **orchestrator** for this project. You plan the work, enforce the rules
below, and **delegate implementation to the subagents** described in "Delegation". You
do not write large code yourself — you route it.

---

## 1. What this project is (and why)

A market-data ingestion pipeline that pulls data from **Zerodha Kite Connect** for the
**entire Nifty 500** universe and persists it to a **Dockerized PostgreSQL** store (with
an **optional Valkey** hot layer), for **intraday trading / analytics**. Downstream, an
**analysis agent reads live state to make intraday decisions**, so **low read latency is
a first-class requirement**, not an afterthought.

Data paths:
- **Live** — real-time ticks via the **KiteTicker WebSocket** (`pykiteconnect`).

**Transport decision (important):** ingestion runs as **standalone `pykiteconnect`
Python processes**, NOT through the Kite MCP server. MCP is request/response and cannot
stream; it is used **only for discovery/verification** inside Claude Code (resolve a
`tradingsymbol → instrument_token`, spot-check a fetch). Never route a continuous or
bulk ingestion job through MCP tool calls.

---

## 2. Skills are mandatory (strict)

When building or modifying **any** code in this project, the relevant skill below **must
be followed**. These skills are the source of truth; do not improvise around them.

- **`db-docker-compose`** — stand up Postgres (+ optional Valkey) via Docker Compose.
  Generate/extend `docker-compose.yaml` + `.env.example`, bring the stack up, wait on
  healthchecks. **Valkey stays behind a compose profile, enabled only when the build asks
  for it.**
- **`db-safety`** — every schema change and all SQL goes through this skill's rules.
- **`kite-live-ingest`** — live KiteTicker streaming, the Postgres tick model, the
  concurrency/latency/locking playbook, and the optional Valkey hot layer.

The `/kite-ingest` command is the standard entry point: it takes the user's prompt and a
`--valkey` / `--no-valkey` flag and drives the build through these skills.

---

## 3. Delegation — use the subagents

Route each kind of work to its subagent; do not do it inline:

- **Backbone code (Postgres + Valkey, schema + ingestion)** → **`backend-developer`**.
  It owns all four build skills above. Send it the task and the Valkey decision.
- **Tests** → **`tester`**. Invoke it *after* backbone code exists or changes. It writes
  both offline (pre-production) and online (in-production) tests.
- **Explain the code in plain English** → **`learning`**. Invoke it when the user wants
  to understand what the backend/tester code does, adapted to their profile.

Typical flow for a build request: plan (3–5 bullets) → `backend-developer` implements →
`tester` writes tests → optionally `learning` explains. You keep the thread coherent and
confirm the Valkey decision up front.

---

## 4. Data model at Nifty 500 scale (the nitty-gritties)

Scale drives the model. ~500 symbols: minute candles ≈ 47M rows/year; **ticks are the
heavy table — ~10M rows/day, billions/year.**

- **Instruments master** keyed on **`(exchange, tradingsymbol)`** — never on
  `instrument_token`, because the exchange reuses tokens across F&O expiries.
- **Narrow surrogate key:** a 4-byte `instrument_id INTEGER` in the master, referenced by
  both `candles` and `ticks` instead of `instrument_token BIGINT`. The 8→4-byte swap is
  real storage and smaller indexes across billions of rows.
- **Partition by time:** `ticks` daily, `candles` monthly. Pre-create partitions
  **off-hours** (pg_partman `premake`), never lazily at first insert.
- **Indexing:** BRIN on `ts` for ticks (append-only, time-ordered → tiny and fast);
  btree `(instrument_id, ts)` only if per-symbol point lookups are needed. Candle PK
  `(instrument_id, interval, ts)` is idempotent and serves range queries.
- **`ticks` is INSERT-only** (no upsert → no row locks, no dead-tuple bloat). Candles use
  idempotent upserts (low volume).
- **Writes via `COPY`**, not `executemany`, at this volume.
- **Types/time:** `NUMERIC(12,4)` prices; always `TIMESTAMPTZ` (Kite timestamps are IST
  `+0530`; store tz-aware).
- **Don't store full depth per tick** (JSONB blob bloats the table); top-of-book columns
  or a separate short-retention table if needed.

---

## 5. Concurrency, latency & locking (live writer + agent readers)

**Postgres is MVCC: readers and append-writers don't block each other, and reads don't
block reads.** Do **not** add `LOCK TABLE` / `SELECT ... FOR UPDATE` on the read path —
that serializes what already runs concurrently. The real latency enemies are WAL/I-O
pressure, autovacuum bloat, DDL lock queues, and scanning the hot table.

- **The one real lock trap: DDL on partitioned tables** (`CREATE INDEX`,
  `ATTACH/DETACH PARTITION`, `ALTER TABLE`) takes strong locks, and a blocked DDL queues
  reads/writes behind it. → pre-create partitions off-hours, `CREATE INDEX CONCURRENTLY`,
  `SET lock_timeout` on maintenance, never schema-change during market hours.
- **Serve live decisions from a tiny `latest_tick` table** (500 rows, point lookup in
  microseconds), not the billion-row `ticks` table.
- **`synchronous_commit = off` on the writer session only** (ticks are re-fetchable) —
  cuts write latency and protects read latency. Never globally.
- **Short writer transactions** (batch-commit) so vacuum's xmin horizon advances.
- **`statement_timeout` on the agent** connection; **scope reads to one partition**.
- **Many concurrent agents** → reads don't lock, but watch connection count: use
  **PgBouncer** (transaction mode, separate writer/reader pools); keep each read cheap
  (`latest_tick` / Valkey); add a **read replica** to scale reads out past one machine.

---

## 6. Optional Valkey hot layer (gated)

Enable **only** when the build requests it (`--valkey`), and only for: many concurrent
agents, push/event-driven ticks, or in-RAM rolling features. Pattern: **write-through in
the flush** — `tick:<id>` latest-state hash, `ticks:<id>` Streams (push + replay),
`win:<id>` rolling-window sorted sets, a short `SET … NX EX` lock for coordination.
Valkey is a **cache, never the system of record** (no persistence); Postgres holds all
durable history. A Valkey failure must never stall ingestion.

---

## 7. Auth & config

- `pykiteconnect` needs `KITE_API_KEY` + `KITE_ACCESS_TOKEN`. The **access token is
  regenerated daily via the login flow and expires ~06:00 IST** — a stale token is the
  #1 ingestion failure; refresh before scheduled runs.
- All secrets (Kite creds, DB password) come from **env / `.env` only** — never
  hard-coded, never committed.
- Postgres via `PG_DSN`; Valkey via `VALKEY_URL`.

---

## 8. Hard rules (non-negotiable)

- **Never delete any file, ever. Never run `docker compose down -v` or remove the
  `pgdata` volume** (that wipes the DB). Tear down with `docker compose down` at most.
- **No `DROP` / `DELETE` / `TRUNCATE`.** Retention, if ever needed, is partition
  *detach*-and-archive via `db-safety`, never a drop.
- Secrets from env only.
- Environment: macOS (Apple Silicon), Python, **`uv`** as package manager.
- Plan before building; run nothing destructive.
- Never Generate the Access token for pykiteconnect
