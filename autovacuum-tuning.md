# 🐘 Autovacuum Configuration — Production-Ready Settings, Not Defaults

**By [Dipak Rijal](https://www.rijal.co.nz)** · Oracle Certified Master · PostgreSQL SME

Postgres ships with autovacuum enabled and set to conservative defaults that work fine for a 10 GB database with light writes. They are absolutely wrong for any high-throughput production workload. This note is the configuration I land on after tuning autovacuum across banking, manufacturing, and government systems.

---

## What autovacuum is actually doing

Autovacuum is a background process that runs VACUUM and ANALYZE automatically when tables cross configurable thresholds of dead tuples or row changes. It is not optional — disable it and your database will eventually fall over from transaction ID wraparound. The question is never *whether*, only *how aggressively*.

---

## The defaults — and why they hurt at scale

The two parameters that drive most autovacuum behaviour:

- `autovacuum_vacuum_threshold` = 50 rows
- `autovacuum_vacuum_scale_factor` = 0.20 (20%)

A table is vacuumed when dead tuples exceed `50 + 0.20 × table_size`. For a 100-million-row table that's 20 million dead tuples before autovacuum even looks at it. By then you've got bloat, performance has degraded, and the vacuum itself takes ages.

---

## The settings that matter

A reasonable starting point for production with moderate-to-high write volume:

```ini
# postgresql.conf
autovacuum = on
autovacuum_max_workers = 5              ; default 3 — most modern hosts handle more
autovacuum_naptime = 30s                ; default 1min — wake up more often
autovacuum_vacuum_scale_factor = 0.05   ; default 0.20 — kick in earlier
autovacuum_analyze_scale_factor = 0.02  ; keep planner statistics fresh
autovacuum_vacuum_cost_limit = 2000     ; default 200 — allow more work per round
autovacuum_vacuum_cost_delay = 10ms     ; adjust pressure on the system
```

The cost-limit / cost-delay pair is autovacuum's throttling mechanism. Raising the cost limit lets each worker do more work before sleeping; raising the delay slows it down. Most production tuning lives in this pair.

---

## Per-table tuning — where the real wins are

Cluster-wide settings are a blunt instrument. The real precision comes from tuning hot tables individually:

```sql
ALTER TABLE high_churn_events SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_analyze_scale_factor = 0.005,
  autovacuum_vacuum_cost_limit = 5000
);
```

Identify candidates from `pg_stat_user_tables`: any table with a high rate of `n_tup_upd + n_tup_del` deserves its own thresholds.

---

## Anti-wraparound vacuums — the one you must not block

Postgres transaction IDs are 32-bit. To avoid running out, autovacuum runs special "freeze" passes on tables. If these get blocked — typically by long-running transactions holding back the xmin horizon — the cluster will eventually enter a read-only safety mode to prevent corruption.

Watch for `(to prevent wraparound)` in `pg_stat_activity`. If you see it, **do not cancel it.** Instead, find the long-running transaction holding things back:

```sql
SELECT pid, age(backend_xmin) AS xmin_age, state, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY age(backend_xmin) DESC
LIMIT 10;
```

Kill the offender, not the vacuum.

---

## Monitoring autovacuum

What ran, and when:

```sql
SELECT relname, last_autovacuum, last_autoanalyze, autovacuum_count, n_dead_tup
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

What is running right now:

```sql
SELECT pid, datname, query, query_start
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%';
```

---

## Common mistakes

- **"Just turn it off, it slows us down."** It doesn't slow you down — bloat does. Disabling autovacuum is the fastest way to a 3am call.
- **Raising `autovacuum_max_workers` to 20 without raising the cost limit.** Workers share the same cost budget by default; more workers each do less work.
- **Identical settings for every table.** A one-million-row reference table and a one-billion-row event log have nothing in common. Tune per-table.
- **Forgetting `autovacuum_analyze_scale_factor`.** Stale statistics make the planner choose bad plans. Analyze keeps queries fast.

---

## The bottom line

Autovacuum is one of the highest-leverage configuration areas in Postgres. The defaults are a starting point for tiny databases, not a target for production. Lower the scale factors, raise the cost limit, identify your top ten high-churn tables, and tune each one individually. Then verify it's working — never assume.

> *"The boring operational stuff — monitoring bloat, tuning autovacuum, right-sizing pools — that's what keeps production alive."*

---

🔧 **Free tools for engineers** at **[rijal.co.nz](https://www.rijal.co.nz)** — SQL formatter, cron translator, and a dozen dev utilities. No login, no paywall.

⭐ Found this useful? Star the [repo](https://github.com/dipak-rijal/dipak-rijal-nz) — contributions welcome.
