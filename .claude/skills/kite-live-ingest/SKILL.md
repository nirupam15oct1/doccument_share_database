---
name: kite-live-ingest-v3
description: >
  Capture live market data from Zerodha Kite Connect and persist it to the
  Dockerized PostgreSQL store. Primary path is the real-time WebSocket
  (pykiteconnect KiteTicker: subscribe + modes ltp/quote/full) streaming ticks
  into a buffered batch writer; secondary path is interval snapshot polling via
  kite.quote / Kite MCP get_quotes when a persistent streamer can't run. Covers
  modes, the parsed tick dict, reconnection, and high-volume batched inserts.
  Defers destructive/schema SQL to the db-safety skill.
---

# Kite live market-data ingestion → PostgreSQL

## What "live" actually means here (read this first)

There are two genuinely different things, and conflating them is the usual mistake:

- **Streaming (ticks) — `KiteTicker` WebSocket. This is the real live path.** One long
  TCP connection pushes every quote update as a binary packet. Up to **3000
  instruments per connection**, up to 3 connections per API key. This is what you want
  for intraday trading/analytics.
  Reference: https://kite.trade/docs/connect/v3/websocket/
- **Snapshots (polling) — `kite.quote` / Kite MCP `get_quotes`.** Request/response.
  You get the state *at the moment you ask*; you miss everything between polls. Fine
  for low-frequency monitoring or when you can't run a persistent process. **MCP only
  does this** — it cannot stream, so don't treat MCP as the live transport for a
  pipeline.

Default to streaming. Use the Kite MCP tools only for discovery (`search_instruments`
to resolve a token) or a quick manual check inside Claude Code.

## Prerequisites

Same as the historical skill: `uv add kiteconnect psycopg[binary]`, Dockerized
Postgres reachable via a `PG_DSN` env var, and credentials in env
(`KITE_API_KEY`, `KITE_ACCESS_TOKEN`; the access token **expires daily ~06:00 IST**).
The `instruments` master table (keyed on `(exchange, tradingsymbol)`) must already be
populated — reuse the one from `kite-historical-ingest`; a streamer subscribes by
numeric `instrument_token` resolved from it.

## Modes — pick the smallest packet that carries what you need

The WebSocket streams in one of three modes (bandwidth grows with detail):

| mode    | packet | contains                                                       |
|---------|--------|----------------------------------------------------------------|
| `ltp`   | 8 B    | last traded price only                                         |
| `quote` | 44 B   | LTP, OHLC, volume, avg price, buy/sell qty (no depth)          |
| `full`  | 184 B  | everything in quote **+ market depth (5+5) + OI + timestamps** |

`pykiteconnect` parses the binary packet into a dict for you — you never touch bytes.
In `full` mode a tick looks like:

```python
{
  'instrument_token': 408065, 'mode': 'full', 'tradable': True,
  'last_price': 1412.95, 'last_traded_quantity': 5, 'average_traded_price': 1412.47,
  'volume_traded': 7360198, 'total_buy_quantity': 0, 'total_sell_quantity': 5191,
  'ohlc': {'open': 1396, 'high': 1421.75, 'low': 1395.55, 'close': 1389.65},
  'change': 1.67, 'last_trade_time': datetime(...), 'exchange_timestamp': datetime(...),
  'oi': 0, 'oi_day_high': 0, 'oi_day_low': 0,
  'depth': {'buy': [{'price':.., 'quantity':.., 'orders':..}, ... x5],
            'sell': [ ... x5 ]}
}
```

(Indices like NIFTY 50 send a reduced packet — LTP + OHLC + change, no depth/volume.)

## Tick table — capture raw, index for time queries

Ticks arrive many-per-second; `exchange_timestamp` is only second-resolution, so two
ticks can share it. Use a surrogate key and index on `(instrument_token, ts)`.

```sql
CREATE TABLE IF NOT EXISTS ticks (
    id               BIGSERIAL   PRIMARY KEY,
    instrument_token BIGINT      NOT NULL,
    ts               TIMESTAMPTZ NOT NULL,          -- exchange_timestamp
    last_price       NUMERIC     NOT NULL,
    last_qty         BIGINT,
    avg_price        NUMERIC,
    volume           BIGINT,
    buy_qty          BIGINT,
    sell_qty         BIGINT,
    oi               BIGINT,
    depth            JSONB                            -- optional, full mode only
);
CREATE INDEX IF NOT EXISTS ticks_token_ts ON ticks (instrument_token, ts);
```

If you only need one row per instrument per second (de-duped), make
`(instrument_token, ts)` the PK and `ON CONFLICT DO NOTHING` instead of a surrogate id.
Plain Postgres has no Timescale compression, so for heavy symbols consider daily
range partitioning on `ts` — but keep that behind the **db-safety** skill.

## Streaming ingestion snippet (buffer → batch flush)

The critical design point: **never insert one row per tick.** Buffer in memory and
flush in batches, or the DB becomes the bottleneck during bursts.

```python
import os, threading, time
from kiteconnect import KiteTicker
import psycopg

conn = psycopg.connect(os.environ["PG_DSN"], autocommit=True)
TOKENS = [408065, 884737]                              # resolve from instruments table

_buf, _lock = [], threading.Lock()

INSERT = """
    INSERT INTO ticks (instrument_token, ts, last_price, last_qty, avg_price,
                       volume, buy_qty, sell_qty, oi)
    VALUES (%s,%s,%s,%s,%s,%s,%s,%s,%s)
"""

def _flush_loop():
    while True:
        time.sleep(1.0)                                # flush at most ~1/sec
        with _lock:
            batch, _buf[:] = _buf[:], []
        if batch:
            with conn.cursor() as cur:
                cur.executemany(INSERT, batch)

def on_ticks(ws, ticks):
    with _lock:
        for t in ticks:
            _buf.append((
                t["instrument_token"], t.get("exchange_timestamp"),
                t["last_price"], t.get("last_traded_quantity"),
                t.get("average_traded_price"), t.get("volume_traded"),
                t.get("total_buy_quantity"), t.get("total_sell_quantity"),
                t.get("oi"),
            ))

def on_connect(ws, response):
    ws.subscribe(TOKENS)
    ws.set_mode(ws.MODE_FULL, TOKENS)                  # or MODE_QUOTE / MODE_LTP

def on_close(ws, code, reason):
    pass  # library auto-reconnects by default; log and let it recover

kws = KiteTicker(os.environ["KITE_API_KEY"], os.environ["KITE_ACCESS_TOKEN"])
kws.on_ticks, kws.on_connect, kws.on_close = on_ticks, on_connect, on_close

threading.Thread(target=_flush_loop, daemon=True).start()
kws.connect()                                          # blocking; threaded=True to not block
```

Operational notes:
- The streamer is a **persistent process**, not a one-shot call. Run it under a
  supervisor (systemd / a loop that restarts on exit) for market hours.
- `KiteTicker` auto-reconnects on drop; subscriptions are re-sent from `on_connect`,
  so keep subscribe/set_mode there so they survive reconnects.
- Market data is binary; text frames on the same socket are order/error/alert
  postbacks — handle via `on_order_update` / `on_error` if you need them, ignore the
  1-byte heartbeats.

## Secondary path — snapshot polling (MCP-aligned)

When a long-running streamer isn't an option, poll. `kite.quote()` accepts up to
**500** `exchange:tradingsymbol` keys (`/quote/ohlc` and `/quote/ltp` allow 1000).
This is exactly what Kite MCP `get_quotes` wraps.

```python
import time
while True:
    q = kite.quote(["NSE:INFY", "NSE:TATAMOTORS"])     # snapshot, not every trade
    rows = [
        (d["instrument_token"], d["timestamp"], d["last_price"],
         d["last_quantity"], d["average_price"], d["volume"],
         d["buy_quantity"], d["sell_quantity"], d.get("oi"))
        for d in q.values()
    ]
    with conn.cursor() as cur:
        cur.executemany(INSERT, rows)
    time.sleep(1)                                      # mind the rate limit
```

Be explicit that this records *samples*, not ticks — fine for a 1–5s monitor, wrong if
you need every trade.

## Data model at Nifty 500 scale — best practices

This is where plain Postgres is actually stressed. Full-mode streaming of ~500 Nifty
500 stocks produces, very roughly:

- ~1 update/sec/instrument × 500 × ~22,500 trading seconds/day ≈ **~10M ticks/day**,
  i.e. **billions/year**. Open/close and volatile names burst several× higher.

Design for that.

**1. Shared surrogate key.** Reference `instruments(instrument_id)` — the same 4-byte
`INTEGER` surrogate the historical skill adds — instead of `instrument_token BIGINT`.
At billions of rows the 4 bytes/row saved is real, and joins to the master stay cheap.

**Why a surrogate, not the token:** across billions of tick rows the 8→4-byte swap
(`BIGINT` → `INTEGER`) is real storage saved and smaller indexes; and
`(exchange, tradingsymbol)` stays the natural key in the master because the exchange
reuses `instrument_token` across F&O expiries, so a token can never be a stable key.

**2. Partition ticks by day (non-negotiable at this volume).**

```sql
CREATE TABLE ticks (
    instrument_id INTEGER       NOT NULL REFERENCES instruments(instrument_id),
    ts            TIMESTAMPTZ   NOT NULL,                 -- exchange_timestamp
    last_price    NUMERIC(12,4) NOT NULL,
    last_qty BIGINT, avg_price NUMERIC(12,4), volume BIGINT,
    buy_qty BIGINT, sell_qty BIGINT, oi BIGINT
) PARTITION BY RANGE (ts);

CREATE TABLE ticks_2026_11_04 PARTITION OF ticks
    FOR VALUES FROM ('2026-11-04') TO ('2026-11-05');
```

Daily partitions keep each child small, make time-range queries touch one or few
partitions, and turn retention into a metadata operation. Automate partition creation
with **pg_partman** — an extension, which fits your stack (you already run
pgvector/pgvectorscale).

**3. Index for how you read, not reflexively.** Ticks are append-only and arrive
time-ordered, so a **BRIN index on `ts`** is tiny (kilobytes) and excellent for
time-range scans — far cheaper than a btree over billions of rows.

```sql
CREATE INDEX ON ticks USING BRIN (ts);
```

Add a btree on `(instrument_id, ts)` *only if* you run per-symbol point lookups;
otherwise skip it and save the write cost and space. Also **drop the `BIGSERIAL id`**
from the earlier minimal table — at this scale it's 8 bytes/row of pure overhead with
no query value; row identity comes from `(instrument_id, ts)` plus insertion order.

**4. Don't store full depth per tick.** The 5+5 JSONB depth blob multiplies table
size several-fold across billions of rows. Keep it out of the tick table; if you need
it, store top-of-book as two columns (`best_bid`, `best_ask`), or write full depth to a
separate short-retention table.

**5. Write with COPY.** Flush each buffered batch via `cursor.copy()` into the current
day's partition — much faster than `executemany` under burst load.

**6. Retention / rollup (optional — respects your no-delete stance).** Raw ticks are
rarely queried raw after the session; analytics live in `candles`. If storage matters,
**detach** old daily partitions and move them to a cold tablespace or archive
(non-destructive) rather than dropping. Any pruning goes through the **db-safety** skill.

**First-principles flag:** this section is essentially hand-building what TimescaleDB
gives for free — hypertables (= your partitioning), continuous aggregates (= your
tick→candle rollup), native compression, and retention policies. Plain Postgres was a
deliberate choice and it works, but if Nifty 500 *tick* capture is a core workload,
that decision is worth revisiting — you're reimplementing the hard parts by hand.

## Concurrency & latency — live writer vs. agent reader

Two clients hit the DB at once: the tick writer inserting continuously, and the agent
running SELECTs to make intraday decisions where milliseconds matter. The instinct is
"manage locking" — in Postgres that's mostly the wrong frame, and coding to it would
*add* latency. The accurate picture:

**Postgres uses MVCC, so readers and writers do not block each other.** Each row carries
version/visibility info; a SELECT reads a consistent snapshot of already-committed rows
while an INSERT writes new versions alongside. An append to `ticks` never waits on the
agent's query, and the agent's query never waits on the insert. So do **not** put
`LOCK TABLE` or `SELECT ... FOR UPDATE` on the read path — that serializes what MVCC
already runs in parallel. The real latency enemies are WAL/I-O pressure, autovacuum
bloat, DDL lock queues, and scanning the hot table. Manage those.

### Specialized segment — database locking: what locks, what doesn't

1. **Reader ↔ append-writer: no lock conflict.** Leave it to MVCC. Adding explicit
   locks is the mistake.
2. **Keep `ticks` INSERT-only — no `ON CONFLICT` upsert.** An upsert takes a row lock on
   conflict and leaves a dead tuple (the superseded version) that autovacuum must later
   clean → bloat → slower scans. Pure inserts have neither. (Upserts are fine on the
   tiny `candles` / `latest_tick` tables; wrong on the billion-row `ticks` table.)
3. **The one real lock trap: DDL on partitioned tables.** `CREATE INDEX`
   (non-concurrent), `ATTACH`/`DETACH PARTITION`, and `ALTER TABLE` take strong locks
   (`SHARE` / `ACCESS EXCLUSIVE`). Worse, a blocked DDL sits at the head of the lock
   queue and **new reads and writes queue behind it** — a brief stall cascades. Fixes:
   - Pre-create partitions **ahead of time, off-hours** (pg_partman `premake`), never
     lazily at first insert.
   - Build indexes with `CREATE INDEX CONCURRENTLY` (no long exclusive lock).
   - Guard every maintenance statement with `SET lock_timeout = '2s'` so it fails fast
     instead of queueing ahead of live traffic.
   - Never run schema changes during market hours.
4. **Keep writer transactions short.** Commit each buffered batch and release. A
   long-open writer transaction pins the vacuum horizon (`xmin`), dead tuples pile up,
   and reads slow. The batch-commit flush loop already does this — don't wrap the whole
   session in one transaction.
5. **Bound the agent's queries.** `SET statement_timeout = '500ms'` on the agent
   connection so a pathological query fails fast instead of piling up connections. A
   query that blows the latency budget is useless for a live decision anyway.

### Latency levers (bigger wins than any lock tuning)

- **Serve live decisions from a tiny "latest state" table, not the hot `ticks` table.**
  The agent rarely needs raw tick history to act — it needs *current price/state per
  symbol*. Maintain a 500-row table the writer upserts; the agent reads it with a point
  lookup (microseconds) instead of scanning billions of rows:
  ```sql
  CREATE TABLE latest_tick (
      instrument_id INTEGER PRIMARY KEY REFERENCES instruments(instrument_id),
      ts TIMESTAMPTZ NOT NULL, last_price NUMERIC(12,4) NOT NULL,
      volume BIGINT, oi BIGINT
  );
  -- in the flush, after appending to ticks:
  INSERT INTO latest_tick (instrument_id, ts, last_price, volume, oi)
  VALUES (%s,%s,%s,%s,%s)
  ON CONFLICT (instrument_id) DO UPDATE SET
      ts=EXCLUDED.ts, last_price=EXCLUDED.last_price,
      volume=EXCLUDED.volume, oi=EXCLUDED.oi;
  ```
  At 500 rows the upsert bloat is negligible — autovacuum keeps up trivially.
- **`synchronous_commit = off` on the writer connection.** Market data is replaceable —
  if a crash loses the last few ms of WAL you re-fetch it — so you needn't wait for disk
  flush on every commit. This cuts write latency and I/O sharply, which directly protects
  read latency. Set it per-session on the writer, not globally.
- **Scope every read to one partition + index.** Filter on the day partition and
  `instrument_id` / `ts` range so the planner prunes to one child and uses BRIN/btree. A
  parent-table query with no time filter scans every partition.
- **Pool connections, separate the pools.** Run PgBouncer (transaction mode) with
  separate pools for writer and agent, so an agent query burst can't starve the ingester
  of connections, or vice versa.
- **If reads are still contended, read from a replica.** Native streaming replication:
  writer → primary, agent → read replica. That physically separates the write-I/O storm
  from the read path — the cleanest decoupling, standard in plain Postgres.

## Optional: Valkey hot layer (in-RAM front for live reads)

Valkey is a store that lives entirely in RAM, so reads/writes are sub-millisecond and it
absorbs huge numbers of simultaneous readers. Use it as a **hot front in front of
Postgres, never as a replacement**: Postgres stays the durable system of record (all
tick/candle history, everything you query for analysis or backtest); Valkey holds only
live, re-derivable state. `uv add valkey`.

**Why Valkey (project standard):** Valkey is the open-source (BSD), Linux Foundation fork
of Redis and speaks the same protocol and commands (hashes, Streams, sorted sets,
`SET … NX EX`), so every pattern below works unchanged. Use the `valkey` Python client
(`import valkey`, `valkey.Valkey.from_url(VALKEY_URL)`), the `valkey/valkey` image, and
`valkey-cli`. Do not add the `redis` client or `redis` image. For tests,
`fakeredis.FakeValkey` is the in-memory fake.

**Turn it on only if one of these actually bites** — otherwise it's just another process
to run and keep in sync:
- **Many concurrent agents** reading live state (connection pressure on Postgres).
- You want **push / event-driven** ticks instead of agents polling.
- Agents need **in-RAM rolling features** (e.g. last-30s VWAP) on every tick.

If none apply, the Postgres `latest_tick` table already covers you.

**Pattern — write-through in the flush.** The writer updates Postgres (durable) *and*
Valkey (hot) from the same batch. A Valkey hiccup must never stall ingestion: Postgres is
authoritative, so log and continue if Valkey write fails.

```python
import json, valkey
r = valkey.Valkey(host="localhost", port=6379, decode_responses=True)

def push_to_valkey(batch):                       # batch: list of tick tuples
    pipe = r.pipeline(transaction=False)        # one round-trip for the whole batch
    for iid, ts, price, last_qty, avg, vol, bq, sq, oi in batch:
        epoch = ts.timestamp()
        # 1) latest state per symbol — fronts latest_tick (point read for agents)
        pipe.hset(f"tick:{iid}", mapping={"price": price, "ts": epoch, "vol": vol, "oi": oi})
        # 2) event stream per symbol — agents consume pushes, with replay + memory cap
        pipe.xadd(f"ticks:{iid}",
                  {"price": price, "ts": epoch, "vol": vol},
                  maxlen=1000, approximate=True)
        # 3) rolling window — sorted set scored by time; member kept unique via ts
        pipe.zadd(f"win:{iid}", {f"{epoch}:{price}": epoch})
        pipe.zremrangebyscore(f"win:{iid}", 0, epoch - 30)   # keep last 30s
    pipe.execute()
```

How agents use each:
- **Latest price** — `HGETALL tick:<id>` (or pipeline many): a microsecond point read,
  the right path for a live decision. This is the Valkey version of the scoreboard.
- **Event-driven ticks** — agents block on `XREAD`/consumer groups on `ticks:<id>` and
  act the instant a tick lands, instead of polling. Streams also let a reconnecting
  agent replay what it missed (plain pub/sub can't).
- **Rolling features** — `ZRANGEBYSCORE win:<id> <now-30> <now>` returns the last 30s of
  prices to compute VWAP/momentum in memory, no scan of the big `ticks` table.
- **Coordination** — a short-lived `SET key val NX EX n` lock lets several agents avoid
  both firing on the same signal.

**Valkey-specific guardrails:**
- It's volatile and a cache — run it in-memory (persistence off/relaxed) since the
  durable copy is in Postgres. Never read history, ranges, or aggregates-over-days from
  Valkey; that's Postgres's job.
- Keep memory bounded: streams with `MAXLEN`, windows trimmed by `ZREMRANGEBYSCORE`,
  latest-state hashes are naturally capped at the universe size (~500 keys).
- Don't block ingestion on Valkey — write-through is best-effort; Postgres is the source
  of truth.

## Guardrails

- No `DROP`/`TRUNCATE`/`DELETE` in this skill; schema/partition work goes through
  **db-safety**.
- Secrets from env / `.env` only.
- Batch writes; cap buffer growth so a DB stall can't exhaust memory (drop-oldest or
  block-and-log if the buffer exceeds a threshold).
