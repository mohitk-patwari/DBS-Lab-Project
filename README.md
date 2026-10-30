# Adaptive Index Advisor

Automates a decision a database administrator normally makes by hand: **which indexes are worth adding when an application slows down as its data grows.**

The system watches a live PostgreSQL workload, works out which queries are both slow *and* frequent, recommends the indexes that would help most, applies them safely, and then measures the result instead of assuming it.

Built on a demo e-commerce application (users, items, orders, cart, checkout) so there is a realistic query workload to analyse.

---

## Why this exists

Adding an index is not free. It speeds up reads, but every index must be updated on every insert and update, so it slows down writes. A DBA has to work out:

1. Which queries are actually costing the most total time
2. Which column would benefit from an index
3. Whether an index already exists
4. Whether the gain is real

This project does all four automatically, and caps the number of indexes so the write cost stays bounded.

---

## How it works

```
PostgreSQL (pg_stat_statements)
        │
        │  every 60s
        ▼
   collector.py ──────────► data/pg_stats.csv
        │
        ▼
recommender_rules.py ─────► data/recommendations.json
        │
        ▼
   FastAPI  /indexes/*  ──► React admin dashboard
        │
        ├── apply:      CREATE INDEX CONCURRENTLY (+ rotation + audit)
        └── comparison: forced seq scan vs index scan
```

### 1. Collect — what the database is actually doing

`pg_stat_statements` is a built-in PostgreSQL extension that records every query the
database has executed, along with how many times it ran (`calls`), total execution
time, mean execution time and rows returned. It is the database's own record of its
own workload.

`backend/collector.py` snapshots the top 1000 queries by total execution time once a
minute and writes them to `data/pg_stats.csv`.

### 2. Recommend — which index would help most

`backend/recommender_rules.py`:

- Parses each query to find the columns used in `WHERE` and `JOIN ... ON` clauses,
  resolving table aliases (`o.user_id` → `orders.user_id`)
- Aggregates call count and execution time per `(table, column)` pair
- Checks `pg_indexes` to skip columns that are already indexed
- **Recommends** a column if it has **50+ calls** and no existing index
- **Ranks by `calls × avg_time_ms`** — a query taking 2s but run once a day matters
  less than one taking 50ms run ten thousand times an hour
- Keeps the **top 3** and writes `data/recommendations.json`

### 3. Apply — safely, and with a hard cap

`POST /indexes/apply` (admin only):

- Defaults to a **dry run**; `force: true` is required to actually create anything
- Uses `CREATE INDEX CONCURRENTLY` so the table is not locked against writes
- Enforces a **3-index rotation cap**: if three system-created indexes already exist,
  the oldest is dropped before the new one is created
- Every create, drop and failure is written to an `index_audit` table with the actor,
  timestamp and reason

The rotation cap is the point of the whole design — it keeps the write-side cost
bounded no matter how many recommendations the analyser produces.

### 4. Measure — prove the gain rather than assume it

`GET /indexes/comparison/{index_name}` runs the same query twice against the same data:

- once with `enable_indexscan = OFF` and `enable_bitmapscan = OFF`, forcing a
  sequential scan
- once normally, letting the planner use the new index

and returns both timings side by side, so the improvement is measured rather than claimed.

---

## Tech stack

| Layer | Technology |
|---|---|
| Database | PostgreSQL 15 with `pg_stat_statements` |
| Backend | Python, FastAPI, psycopg2, SQLAlchemy, pandas, APScheduler |
| Frontend | React, TypeScript, Vite, TailwindCSS |
| Auth | JWT with role-based access (index endpoints are admin-only) |
| Infra | Docker Compose |

---

## Benchmark dataset

Seeded by `seed_db.py`:

| Table | Rows |
|---|---|
| `users` | 5,000 |
| `items` | 200 |
| `orders` | 20,000 |
| `order_items` | ~40,000 (1–3 line items per order) |

---

## Running it

**Prerequisites:** Docker, Python 3.10+, Node 18+

**1. Start PostgreSQL**

```bash
cd app/data/infra
docker compose up -d
```

`pg_stat_statements` is enabled via `shared_preload_libraries` in the compose file.
Postgres is exposed on `localhost:5432` with database, user and password all set to `demo`.

**2. Install Python dependencies**

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

**3. Seed the database**

```bash
python app/data/infra/seed_db.py
```

**4. Generate workload** (separate terminal)

```bash
python app/workload.py
```

**5. Collect query statistics** (separate terminal)

```bash
python backend/collector.py          # runs every 60s
RUN_ONCE=true python backend/collector.py   # single snapshot
```

**6. Build recommendations**

```bash
python backend/recommender_rules.py
```

**7. Start the API and dashboard**

```bash
uvicorn backend.app.main:app --reload
cd frontend && npm install && npm run dev
```

---

## API

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/indexes/recommendations` | Current ranked index recommendations |
| `GET` | `/indexes/list` | Indexes created by the system |
| `POST` | `/indexes/apply` | Dry run, or create with `force: true` |
| `GET` | `/indexes/comparison/{index_name}` | Seq scan vs index scan timings |

All index endpoints require an admin role.

---

## Design decisions worth noting

**Why cap at three indexes?** Every index slows down writes. Without a cap, an
automated recommender would happily fix read latency and quietly wreck insert
performance. Rotation keeps the trade-off bounded.

**Why `CREATE INDEX CONCURRENTLY`?** A plain `CREATE INDEX` takes a lock that blocks
writes to the table. `CONCURRENTLY` is slower but does not block an application that
is still serving traffic.

**Why rank by `calls × avg_time`?** Ranking by slowest query alone optimises for
outliers. Ranking by call count alone optimises for cheap queries. The product
approximates total time actually spent.

**Why force a sequential scan for the comparison?** Once an index exists the planner
will use it, so there is no way to observe the "before" case without disabling index
scans for that one query. This makes the comparison honest rather than theoretical.

---

## Known limitations

- Candidate extraction is regex-based, so unusual SQL (subqueries, CTEs, functions in
  predicates) may be missed. An AST-based parser would be more robust.
- Only single-column indexes are recommended; composite indexes are not considered.
- The 50-call threshold is a fixed heuristic rather than something derived from the
  workload.
- The comparison measures a representative query per index, not the full workload.
