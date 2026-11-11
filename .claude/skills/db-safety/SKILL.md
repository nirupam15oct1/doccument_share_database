---
name: db-safety
description: Database security and agent-safety rules for this project. Use whenever creating or modifying databases, schemas, roles, GRANT/REVOKE, Row-Level Security, or any code that opens a database connection — including building the Kite ingestion DB or the agent-memory store.
---

# Database Security & Agent-Safety Guidelines

## When to apply
Apply this whenever creating or modifying: databases, schemas, roles,
`GRANT`/`REVOKE`, Row-Level Security, or any code that opens a database
connection. This is the enforced security layer for every database this project
builds. Access control is done by the **Postgres engine** via roles and schemas
— never by trusting the application or the agent to "behave."

## First principle: two different walls
1. **Sandbox (process wall)** — limits what the agent's *code* can touch on the
   machine (filesystem, network, syscalls).
2. **Database role (query wall)** — limits what any *SQL* on a connection may do,
   regardless of what code emitted it.

The realistic threat is not the sandbox breaking; it is the agent being
prompt-injected or buggy and emitting SQL that reads or destroys data it should
not. The database role is the backstop that holds **even if the agent is fully
compromised**, because the engine refuses the query. Design so that no single
leaked prompt can exceed the connection's granted privileges.

## Role model (non-negotiable)
- Each component connects as its own dedicated LOGIN role:
  `NOSUPERUSER NOCREATEDB NOCREATEROLE`.
- The application/agent role is **never the table owner** and **never a
  superuser** — owners and superusers bypass Row-Level Security.
- Grant the minimum verbs: usually `SELECT` (+ `INSERT`/`UPDATE` where it
  writes). Do **not** grant `DELETE` (project data/file-safety stance) or any
  DDL to app roles.
- **Deny by default:** a role sees only the schemas you explicitly grant.
  Everything else is invisible by omission.

## The two gates to reach a table
A role needs BOTH to read/write a table:
1. `USAGE` on the **schema** ("enter the folder").
2. `SELECT`/`INSERT`/… on the **table**.

Miss either gate → denied. "Confine to one schema" = grant both gates on that
one schema, and ensure neither gate is open anywhere else.

## The PUBLIC leak (must revoke)
Every role is automatically a member of the built-in `PUBLIC` group. Some
default grants (especially on schema `public`) go to `PUBLIC`, so *"I granted
nothing"* is **not** the same as *"no access."* Explicitly revoke on every
schema an app role must not reach:

```sql
REVOKE ALL ON SCHEMA public, compliance, risk, staging FROM PUBLIC;
REVOKE CREATE ON SCHEMA public FROM PUBLIC;   -- stop object-planting in public
```

## Recipe: confine a role to exactly one schema
Example — an agent allowed only in `agent_mem`:

```sql
CREATE ROLE agent_login LOGIN PASSWORD :'pw'
    NOSUPERUSER NOCREATEDB NOCREATEROLE CONNECTION LIMIT 5;

-- close PUBLIC leaks / strip anything on forbidden schemas
REVOKE ALL ON SCHEMA public, compliance, risk, staging FROM PUBLIC;
REVOKE ALL ON SCHEMA compliance, risk, staging FROM agent_login;

-- grant exactly one schema, two gates, no DELETE / no DDL
GRANT USAGE ON SCHEMA agent_mem TO agent_login;                       -- gate 1
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA agent_mem
      TO agent_login;                                                 -- gate 2
ALTER DEFAULT PRIVILEGES IN SCHEMA agent_mem
      GRANT SELECT, INSERT, UPDATE ON TABLES TO agent_login;          -- future tables

-- block object creation even in its own schema (privilege-escalation vector)
REVOKE CREATE ON SCHEMA agent_mem FROM agent_login;

-- pin name resolution: bare names can only resolve inside agent_mem
ALTER ROLE agent_login SET search_path = agent_mem;
```

Result: bare `SELECT ... FROM sar_filings` resolves only in `agent_mem`, finds
nothing, errors; a fully-qualified `compliance.sar_filings` fails gate 1 (no
`USAGE`). Both paths closed.

## Why pin search_path
Bare table names resolve via `search_path` (default `"$user", public`). An actor
who can create an object in a schema on that path can **shadow** a trusted object
(search_path hijacking). Pin each role's `search_path` to only its allowed
schema(s), and prefer fully-qualified `schema.table` names in code — this also
removes ambiguity about *which* object a query actually touched (audit value).

## Per-row isolation (optional, for shared schemas): Row-Level Security
If several principals share one table but each must see only its own rows:

```sql
ALTER TABLE agent_mem.agent_memory ENABLE ROW LEVEL SECURITY;
CREATE POLICY own_rows ON agent_mem.agent_memory
    USING (agent_id = current_setting('app.agent_id', true));
```

The app sets `app.agent_id` per session/transaction. **Caveat:** RLS is bypassed
by the table **owner** and by superusers — which is exactly why the app role must
be neither.

## Analyst / human roles: deny-by-default does the work
To keep a junior analyst out of `agent_mem`, simply never grant them `USAGE` on
it. Grant only the schemas they analyze, read-only:

```sql
CREATE ROLE analyst_login LOGIN PASSWORD :'pw' NOSUPERUSER;
GRANT USAGE ON SCHEMA market_derived, reference TO analyst_login;
GRANT SELECT ON ALL TABLES IN SCHEMA market_derived, reference TO analyst_login;
ALTER DEFAULT PRIVILEGES IN SCHEMA market_derived, reference
      GRANT SELECT ON TABLES TO analyst_login;
ALTER ROLE analyst_login SET default_transaction_read_only = on;
ALTER ROLE analyst_login SET search_path = market_derived, reference;
```

Agent and analyst are now **disjoint by allow-list**: neither can reach the
other's schema, and neither reaches compliance unless explicitly added.

## Soft wall vs hard wall (choose by threat model)
- **Schema grants = soft wall** inside one database. They reliably stop *data*
  access, but a confined role can still read system catalogs and learn that
  other schemas/tables *exist* (metadata, not data).
- If even the **existence** of a store must be hidden (regulated PII, sensitive
  agent memory), use a **separate database** or instance — no single SQL query
  crosses a database boundary. Database = opaque wall; schema = see-through wall.

## Credential hygiene (the real perimeter)
All of the above collapses if the agent can read a high-privilege connection
string. The agent process receives **only** its scoped role's DSN, from
env/secrets, never hardcoded, never with a superuser DSN sitting nearby. Rotate
the Kite `access_token` and DB passwords; keep secrets out of logs and the repo.

## Resource guards (availability is safety)
```sql
ALTER ROLE agent_login SET statement_timeout = '15s';
-- CONNECTION LIMIT set at role creation
```
Stops a runaway or injected agent from exhausting the DB even within its allowed
schema.

## Connection-pooler caveat
PgBouncer in transaction mode can reset session settings between pooled
connections. Set `search_path`, RLS variables (`app.agent_id`), and read-only
flags **per-transaction**, not once per session, or they will not stick.

## Pre-flight checklist — satisfy ALL before a database is "done"
- [ ] App/agent role is `NOSUPERUSER NOCREATEDB NOCREATEROLE` and **not** the
      table owner.
- [ ] `REVOKE ... FROM PUBLIC` run on every schema the role must not reach;
      `REVOKE CREATE ON SCHEMA public FROM PUBLIC`.
- [ ] Role granted `USAGE` + minimal table privileges on **exactly** its allowed
      schema(s); nothing on others.
- [ ] No `DELETE`, no DDL granted to app roles. Migrations additive only.
- [ ] `ALTER DEFAULT PRIVILEGES` set so future tables inherit intended grants.
- [ ] `search_path` pinned per role; code uses fully-qualified names.
- [ ] `REVOKE CREATE` on the role's own schema.
- [ ] RLS enabled + policy present where rows of different principals share a
      table; verified owner ≠ app role.
- [ ] Analyst/human roles granted only their analytic schemas, read-only;
      confirmed **no** `USAGE` on `agent_mem` / compliance.
- [ ] Secrets only via env/secrets store; no superuser DSN reachable by the
      agent.
- [ ] `statement_timeout` + `CONNECTION LIMIT` set.
- [ ] Soft wall (schema) vs hard wall (separate DB) decided per sensitivity;
      sensitive stores isolated accordingly.
