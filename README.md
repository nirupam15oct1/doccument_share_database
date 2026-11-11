# Database_LinkdinPost — Kite → Postgres (+ optional Valkey) market-data pipeline

A reference repo showing how to build a **production-grade live market-data store** for
agentic intraday trading on the NSE: ticks streamed from **Zerodha Kite Connect** into a
**Dockerized PostgreSQL** (cold, system of record) with an **optional Valkey/Redis hot
layer** (in-RAM, sub-millisecond reads for agents).

The point of this repo is **not** the trading logic. It is a worked example of a claim:
in Claude Code, a *simple prompt* gets you a throwaway script, but a well-written **skill +
subagent setup** gets you an **engineered database** that holds up at Nifty-500 scale.

---

## 1. What's in here

```
CLAUDE.md                  orchestrator rules — the "brain" that plans and routes work
.claude/
  commands/kite-ingest.md  /kite-ingest — the single entry point for a build request
  skills/
    db-docker-compose/     stand up Postgres (+ Valkey) via Docker Compose
    db-safety/             every schema change & SQL statement obeys these rules
    kite-live-ingest/      live KiteTicker streaming, tick model, latency/locking, Valkey
  agents/
    backend-developer.md   writes the schema + ingestion code (owns the build skills)
    tester.md              writes offline + live tests
    learning.md            explains the code in plain English (placeholder — see §5)
```

How the pieces relate: **CLAUDE.md plans → `/kite-ingest` command drives the build →
`backend-developer` writes code against the skills → `tester` tests it → `learning`
explains it.** You talk to the orchestrator; it routes to the subagents.

---

## 2. Prerequisites (one-time)

- **Claude Code** open in this folder (so it reads `CLAUDE.md`, the command, skills, agents).
- **Docker Desktop** running (for the Postgres / Valkey containers).
- **Python + `uv`** (project standard; macOS / Apple Silicon).
- **Kite credentials** in a local `.env` (never committed):
  ```
  KITE_API_KEY=...
  KITE_ACCESS_TOKEN=...        # regenerate daily via the login flow; expires ~06:00 IST
  PG_DSN=postgresql://...
  VALKEY_URL=redis://localhost:6379   # only if using the hot layer
  ```

---

## 3. How to GENERATE and WRITE the code

The whole build runs through **one command**: `/kite-ingest`. You give it a plain-English
task and a Valkey flag; it does the planning, picks the right skills, and delegates the
actual writing to the `backend-developer` subagent.

### Pattern
```
/kite-ingest [--valkey | --no-valkey] <what to build>
```

### Step by step
1. **Open Claude Code in this folder.** It auto-loads `CLAUDE.md` (the rules) and makes the
   command, skills, and agents available.
2. **Decide the hot layer up front:**
   - `--no-valkey` (default) → live reads served from a Postgres `latest_tick` table.
   - `--valkey` → add the in-RAM Valkey hot layer (hash + streams + sorted sets).
3. **Issue the build request.** Examples:
   ```
   /kite-ingest --no-valkey  create the Postgres schema for Nifty 500 ticks + candles
   /kite-ingest --no-valkey  write the KiteTicker streamer that batch-writes ticks via COPY
   /kite-ingest --valkey     add the write-through Valkey hot layer for live agent reads
   ```
4. **Review the 3–5 bullet plan** the orchestrator states before any code is written, then
   let `backend-developer` implement it against the skills.
5. **Write the tests.** After backbone code exists:
   ```
   use the tester agent to write offline + live tests for the streamer
   ```
6. **Stand up the stack** when code needs a running DB (the `db-docker-compose` skill drives this):
   ```
   docker compose up -d                       # Postgres only
   docker compose --profile valkey up -d      # Postgres + Valkey
   ```

### Rules the generated code always follows (enforced by CLAUDE.md + skills)
- Ingestion is **standalone `pykiteconnect`**, never the MCP server (MCP = discovery only).
- **Nifty-500 data model:** `instrument_id` surrogate key, day-partitioned `ticks`
  (INSERT-only), month-partitioned `candles` (idempotent upsert), BRIN on `ts`, `COPY` writes,
  `NUMERIC(12,4)`, `TIMESTAMPTZ` (IST).
- **Concurrency:** MVCC — no read-path locks; serve live decisions from `latest_tick`;
  `synchronous_commit=off` on the writer session only; pre-create partitions off-hours.
- **Valkey is a cache, never the source of record.** A Valkey failure must not stall ingestion.
- Secrets from `.env` only. **No `DROP` / `DELETE` / `TRUNCATE`; never `docker compose down -v`;
  never delete files.**

---

## 4. The agents (what to call, and when)

| Agent | Use it for | Does NOT |
|-------|-----------|----------|
| `backend-developer` | schema + ingestion + Valkey code | write tests or explanations |
| `tester` | offline (pre-prod correctness) + online (security/latency/connections) tests | change production code |
| `learning` | explain *what the code does and why*, in plain English | modify code |

---

## 5. Learning

> **Placeholder — material to be added.**
>
> The `learning` agent currently runs on a short brief: explain the code's logic for a
> reader with a strong stats / data-science background but no formal CS training — "why"
> before "how", one concrete everyday example per CS/DB concept (partitions, MVCC, indexes,
> Valkey, locking).
>
> A fuller learning skill will replace this brief. _(Add material here.)_
