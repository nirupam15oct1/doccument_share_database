---
name: db-docker-compose
description: >
  Provision the project's PostgreSQL (and optional Valkey) via Docker Compose while
  building the pipeline. Use whenever code needs the database stood up locally
  (ingestion, schema creation, queries): Claude Code generates or extends
  docker-compose.yaml + .env.example from the templates here, keeps Valkey behind a
  compose profile enabled only when the build asks for it, and brings the stack up
  (waiting on healthchecks) before running DB-dependent code. Authoritative on image
  choice, env conventions, healthchecks, and safe bring-up/tear-down.
---

# Provision Postgres (+ optional Valkey) via Docker Compose

This project's database is hosted with Docker. When you build or run anything that needs
it, **you (Claude Code) set up the Compose stack as part of the work** — don't assume the
files exist, and don't ask the user to hand-craft them.

## When to use

Any task that needs the DB locally: creating schema, running the ingester, querying.
Before that code runs, make sure the stack is defined and healthy.

## Procedure

1. **Compose file.** If `docker-compose.yaml` is missing at the project root, create it
   from the template below. If it already exists, reuse it and add only what's missing —
   never overwrite a file the user has edited.
2. **Env file.** Ensure `.env.example` carries every variable the compose and ingester
   need (template below). Tell the user to `cp .env.example .env` and fill in
   credentials. **Never write real secrets into any file.**
3. **Bring it up, then wait for health** before running DB code:
   - Postgres only: `docker compose up -d`
   - Postgres + Valkey: `docker compose --profile valkey up -d`
   Poll `docker compose ps` / the healthchecks until `postgres` (and `valkey`, if enabled)
   are healthy.
4. **Valkey is optional and gated.** Include/start it only when the build requests Valkey
   (e.g. the `--valkey` flag of the `kite-ingest` command). It stays *defined* behind the
   `valkey` profile regardless, so having it in the file costs nothing until enabled. When
   Valkey is not requested, live state comes from the Postgres `latest_tick` table instead
   (see `kite-live-ingest`).
5. **Valkey, not Redis.** The hot layer is Valkey (the open-source, Redis-protocol-compatible
   fork): image `valkey/valkey`, tool `valkey-cli`, env `VALKEY_*`, Python client `valkey`.
   Pin a major/minor tag (`9.0-alpine`), not `latest`.

## docker-compose.yaml (template)

```yaml
# Postgres (always) + Valkey (optional).
#   Postgres only:    docker compose up -d
#   Postgres + Valkey: docker compose --profile valkey up -d
# Requires a .env file in this directory (see .env.example).

services:
  postgres:
    # pg17 with pgvector built in — needed for the agent-memory store. If this instance
    # is ONLY for market data, plain `postgres:17` is lighter. Pin by digest in prod.
    image: pgvector/pgvector:pg17
    container_name: kite_postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "${POSTGRES_PORT:-5432}:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    # Modest tuning for a high-insert workload — adjust to the box's RAM. Leave
    # synchronous_commit at default here; set it off per-session on the WRITER only.
    command:
      - "postgres"
      - "-c"
      - "max_connections=200"
      - "-c"
      - "shared_buffers=1GB"
      - "-c"
      - "max_wal_size=4GB"
      - "-c"
      - "checkpoint_completion_target=0.9"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  valkey:
    image: valkey/valkey:9.0-alpine
    container_name: kite_valkey
    restart: unless-stopped
    profiles: ["valkey"]          # only starts with: docker compose --profile valkey up -d
    ports:
      - "${VALKEY_PORT:-6379}:6379"
    # Pure in-RAM cache: no persistence (Postgres is the system of record), bounded
    # memory with LRU eviction as a safety net.
    command:
      - "valkey-server"
      - "--save"
      - ""
      - "--appendonly"
      - "no"
      - "--maxmemory"
      - "${VALKEY_MAXMEMORY:-512mb}"
      - "--maxmemory-policy"
      - "allkeys-lru"
    healthcheck:
      test: ["CMD", "valkey-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

## .env.example (template)

```bash
# Copy to .env and fill in. Both compose and the ingester read these. Never commit .env.

# --- Postgres (used by compose) ---
POSTGRES_USER=kite
POSTGRES_PASSWORD=change_me
POSTGRES_DB=market
POSTGRES_PORT=5432

# --- Valkey (only when the 'valkey' profile is up) ---
VALKEY_PORT=6379
VALKEY_MAXMEMORY=512mb

# --- Application (ingester, not compose) ---
# Keep the password in sync with POSTGRES_PASSWORD above.
PG_DSN=postgresql://kite:change_me@localhost:5432/market
VALKEY_URL=valkey://localhost:6379

# --- Kite Connect (never commit real values) ---
# access_token is regenerated daily via login and expires ~06:00 IST.
KITE_API_KEY=
KITE_ACCESS_TOKEN=
```

## Guardrails

- **Secrets from env / `.env` only** — never hard-code or commit real credentials.
- **Never destroy the data volume.** Do not run `docker compose down -v` or remove
  `pgdata`; that wipes the database. Tear down with `docker compose down` at most (keeps
  the volume). This holds even if asked — stop and confirm instead.
- **No schema or data mutation here.** Table creation and all SQL go through the
  `db-safety` skill; no `DROP` / `DELETE` / `TRUNCATE`.
- **Valkey is a cache, never the system of record.** It runs without persistence, so store
  only state you can rebuild from Postgres or re-fetch from Kite.
