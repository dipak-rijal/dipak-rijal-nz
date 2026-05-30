# 🐘 Bloat Detection & Vacuum Tuning — A Practical Field Guide

**By [Dipak Rijal](https://www.rijal.co.nz)** · Oracle Certified Master · PostgreSQL SME

If your once-snappy Postgres queries have quietly become sluggish over months — same data shape, same indexes, same hardware — bloat is almost always the answer. This is the note I wish every team had before they reached for `VACUUM FULL` and locked their primary table for an hour.

---

## What bloat actually is

Postgres uses MVCC (Multi-Version Concurrency Control). When you UPDATE or DELETE a row, the original isn't overwritten — it's marked as dead, and a new version is written. Readers can keep using the old version through consistent snapshots, while writers move on. Brilliant for concurrency.

The bill comes later. Dead tuples accumulate. Tables grow larger than the live data they hold. Indexes grow with them. Disk reads pull mostly-dead pages into shared buffers. Queries slow down — not because they're badly written, but because they're shoveling through ghosts.

This is bloat. And it's normal — within limits.

---

## How to find it

Postgres ships with `pgstattuple`, an extension that gives precise (but expensive) measurements:

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('public.orders');
```

For day-to-day monitoring, the cheaper estimate is more practical:

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       round(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 1) AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

A healthy table typically sits below 10–20% dead tuples. Sustained 30%+ is a problem. Sustained 50%+ is a crisis already in motion.

---

## The vacuum family — three different tools

### Plain VACUUM
Marks dead tuples reusable for future inserts. Does **not** return space to the operating system. Non-blocking. This is the safe, default tool.

### VACUUM FULL
Rewrites the entire table into a new file, discarding dead space. Returns disk back to the OS. **Takes an ACCESS EXCLUSIVE lock for its entire run** — readers and writers blocked. Almost never the right answer on production.

### pg_repack / pg_squeeze
Third-party extensions that achieve what VACUUM FULL does, without the table lock. If you genuinely need to reclaim space on a live system, this is the tool.

---

## Index bloat — the silent twin

Tables get the attention; indexes do the damage. Indexes bloat through different mechanisms — page splits, deletes that don't immediately recycle pages — and a bloated index can be three to five times its healthy size.

```sql
SELECT schemaname, relname, indexrelname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC
LIMIT 20;
```

The fix:

```sql
REINDEX INDEX CONCURRENTLY idx_orders_customer_id;
```

`CONCURRENTLY` builds a fresh copy without blocking writes, then swaps it in. Available since Postgres 12. Use it.

---

## When to intervene manually

Most of the time, autovacuum should handle this for you (see the [autovacuum tuning note](https://github.com/dipak-rijal/dipak-rijal-nz/blob/main/autovacuum-tuning.md)). You step in manually when:

- A bulk delete just removed millions of rows — kick off a manual VACUUM rather than wait.
- Anti-wraparound vacuums are queueing up (you'll see `(to prevent wraparound)` in `pg_stat_activity`).
- Autovacuum is being blocked by a long-running transaction — find and kill the offender, not the vacuum.
- A specific high-churn table needs more aggressive thresholds than the cluster default.

---

## The bottom line

Bloat is not a bug — it's MVCC's natural cost. The job is keeping it within tolerance, not eliminating it. Monitor dead-tuple percentage, REINDEX CONCURRENTLY when index bloat creeps up, and reach for VACUUM FULL only when you understand exactly what it will lock and for how long.

> *"The boring operational stuff — monitoring bloat, tuning autovacuum, right-sizing pools — that's what keeps production alive."*

---

🔧 **Free tools for engineers** at **[rijal.co.nz](https://www.rijal.co.nz)** — SQL formatter, cron translator, and a dozen dev utilities. No login, no paywall.

⭐ Found this useful? Star the [repo](https://github.com/dipak-rijal/dipak-rijal-nz) — contributions welcome.
