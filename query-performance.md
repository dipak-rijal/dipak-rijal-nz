# 🐘 Query Performance Checklist — A Systematic Approach

**By [Dipak Rijal](https://www.rijal.co.nz)** · Oracle Certified Master · PostgreSQL SME

Most query tuning starts with the wrong question: "why is *this* query slow?" That assumes you already know which query matters. The systematic approach starts higher up — find what's actually costing you, then tune it. This is the checklist I work through, in order, every time.

---

## Step 1 — install pg_stat_statements (if you haven't)

You cannot tune what you cannot see. `pg_stat_statements` aggregates execution statistics across every query in the cluster — total time, mean time, calls, rows. It is the single most useful extension in Postgres.

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

Restart, then:

```sql
CREATE EXTENSION pg_stat_statements;
```

Now the question changes from "is this query slow?" to "which queries are eating my database?":

```sql
SELECT calls, total_exec_time, mean_exec_time, rows, query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

Sort by `total_exec_time`, not `mean_exec_time`. A query that runs in 5ms but is called a million times an hour matters more than one that takes 30 seconds once a day.

---

## Step 2 — EXPLAIN ANALYZE, the right way

For your top offenders:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT ...;
```

`BUFFERS` is the critical addition almost everyone forgets. It tells you how many pages were read from cache versus disk. A query reading 100,000 pages from disk has a very different fix from one reading 100,000 from cache.

Things to look for in the plan:

- **Seq Scan on a large table** when you expected an index.
- **Rows: estimated 1, actual 1,000,000** — statistics are stale, run ANALYZE.
- **Sort spilling to disk** (`external merge Disk`) — `work_mem` is too low.
- **Nested Loop with high iterations** — the planner picked the wrong join type, usually a stats problem.

---

## Step 3 — check the index situation

**Missing indexes** — find sequential scans on large tables:

```sql
SELECT schemaname, relname, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC
LIMIT 20;
```

**Unused indexes** — each one slows down every INSERT/UPDATE for no benefit:

```sql
SELECT schemaname, relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;
```

Be careful with that second one — `idx_scan = 0` since the last stats reset isn't proof. Indexes supporting weekly or monthly jobs will look unused most of the time.

---

## Step 4 — keep statistics fresh

The planner makes decisions based on `pg_statistic`. If your data shape has shifted since the last ANALYZE, plans go sideways.

```sql
ANALYZE my_table;
```

For columns that need finer-grained statistics:

```sql
ALTER TABLE my_table ALTER COLUMN status SET STATISTICS 1000;
ANALYZE my_table;
```

The default `default_statistics_target` is 100. Bumping it to 500–1000 for skewed columns often fixes mysterious bad plans.

---

## Step 5 — rule out infrastructure issues

Before assuming the query is wrong, confirm the database isn't being starved:

- **Bloat** — covered in the [bloat & vacuum note](https://github.com/dipak-rijal/dipak-rijal-nz/blob/main/bloat-and-vacuum.md). A bloated index can turn a 10ms query into a 2-second query without anything in the query itself changing.
- **Connections** — too many active sessions means too many backends competing. See the [PgBouncer note](https://github.com/dipak-rijal/dipak-rijal-nz/blob/main/connection-pooling-pgbouncer.md).
- **`work_mem` too low** — sorts and hash joins spill to disk. Bump it per-session for known heavy queries: `SET work_mem = '256MB';`.
- **I/O** — `pg_stat_io` (Postgres 16+) or `iostat` on the host. Sometimes the database is fine and the storage isn't.

---

## Step 6 — tune the query itself

This is where most people start — and now you've done the diagnostic work to know whether tuning the SQL is even the right intervention.

Common rewrites:

- Replace `OR` conditions with `UNION ALL` (the planner often handles them better).
- Replace correlated subqueries with `LATERAL` joins.
- Add covering indexes (`INCLUDE (...)`) to enable index-only scans.
- Push aggregations down where possible; pre-aggregate into a materialised view if a heavy report is run repeatedly.

---

## The bottom line

Performance work is detective work. Skipping the diagnostic steps to jump straight to "rewrite the query" wastes hours. `pg_stat_statements` tells you what to look at. `EXPLAIN (ANALYZE, BUFFERS)` tells you why it's slow. Index, statistics, infrastructure, query — in that order.

> *"The boring operational stuff — monitoring bloat, tuning autovacuum, profiling queries — that's what keeps production alive."*

---

🔧 **Free tools for engineers** at **[rijal.co.nz](https://www.rijal.co.nz)** — SQL formatter, cron translator, and a dozen dev utilities. No login, no paywall.

⭐ Found this useful? Star the [repo](https://github.com/dipak-rijal/dipak-rijal-nz) — contributions welcome.
