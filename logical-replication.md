# 🐘 Logical Replication Troubleshooting — Slots, Drift & Lag

**By [Dipak Rijal](https://www.rijal.co.nz)** · Oracle Certified Master · PostgreSQL SME

Logical replication is one of Postgres's most powerful features and one of its most operationally tricky. This note covers the three problems that show up in production: replication slot bloat, publication drift, and decoding lag — and what to do when each one bites.

---

## A quick mental model

Logical replication works through three pieces: a **publication** on the source (what to send), a **subscription** on the target (what to receive), and a **replication slot** that tracks how far the target has caught up. WAL is decoded into row-level changes and streamed.

Different from physical (streaming) replication: logical is row-level not block-level, can cross major versions, can replicate a subset of tables, and tolerates a different shape on the target. It's the only replication that lets you replicate INTO a database that doesn't look identical.

---

## Problem 1 — replication slot bloat

The single most common production incident with logical replication. A slot holds back WAL until the subscriber confirms receipt. If the subscriber falls behind, dies, or is forgotten about, WAL piles up on the source — sometimes filling the disk.

**Detect it:**

```sql
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(
         pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
       ) AS retained_wal
FROM pg_replication_slots
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;
```

If `retained_wal` is in GB and growing, something is wrong.

**Fix it:** First, find out why the subscriber is behind. Network problem? Subscriber down? A conflict it can't resolve? Then either fix the subscriber or — if the slot is genuinely abandoned — drop it:

```sql
SELECT pg_drop_replication_slot('orphaned_slot_name');
```

**Prevent it:** Set `max_slot_wal_keep_size` (Postgres 13+) on the source. The slot will be invalidated once it retains more than this much WAL, protecting the source from filling up. You'll have to rebuild the subscription if that happens, but that's far better than a primary outage.

```ini
max_slot_wal_keep_size = 50GB
```

---

## Problem 2 — publication drift

Logical replication does **not** replicate DDL. Add a column on the source and the subscriber doesn't get it. An INSERT including the new column then fails on the subscriber and replication halts.

**Detect it:** Look in the subscriber's logs for messages like `logical replication target relation "schema.table" is missing some replicated columns`.

**Fix it:** The order matters.

For an **ADD COLUMN**:
1. Add the column on the subscriber first (nullable, or with a default).
2. Then add it on the publisher.
3. Replication resumes.

For a **DROP COLUMN**, reverse the order — drop from the publisher first, then the subscriber.

**Prevent it:** Treat schema migrations as a coordinated operation across both sides. Build it into the deployment pipeline; don't trust a process that says "remember to also run it on the replica."

---

## Problem 3 — decoding lag

Logical decoding is single-threaded per slot, and large transactions are decoded entirely into memory before being sent. A bulk DELETE of 50 million rows can cause apparent "lag" while the source is just busy decoding it.

**Detect it:** Compare `pg_current_wal_lsn()` on the source with `confirmed_flush_lsn` from `pg_replication_slots`. If the gap grows during big transactions but recovers afterwards, that's decoding lag, not replication failure.

**Fix it:**
- **Postgres 14+:** Enable `streaming = on` in the subscription. Large in-progress transactions are streamed to the subscriber rather than buffered to completion on the source. A massive win for bulk operations.
- **Split big DML** at the application level. A million-row delete becomes a hundred ten-thousand-row deletes, each committing in turn.

---

## A few hard-won rules

- Always set `wal_level = logical` on the source before you need it. Changing it requires a restart.
- Always have monitoring on `pg_replication_slots` that alerts on retained WAL — before the disk fills.
- Always include the slot name in your runbook. "There's some slot somewhere" is not a runbook.
- Set `max_slot_wal_keep_size`. Always. The trade-off (rebuild subscription vs primary disk full) is not actually a trade-off.

---

## The bottom line

Logical replication is fantastic when it works and brutal when it doesn't. The three problems above account for the great majority of incidents. Build monitoring around them, treat schema changes as cross-cluster operations, and let `max_slot_wal_keep_size` protect you from yourself.

> *"The boring operational stuff — monitoring bloat, tuning autovacuum, watching replication slots — that's what keeps production alive."*

---

🔧 **Free tools for engineers** at **[rijal.co.nz](https://www.rijal.co.nz)** — SQL formatter, cron translator, and a dozen dev utilities. No login, no paywall.

⭐ Found this useful? Star the [repo](https://github.com/dipak-rijal/dipak-rijal-nz) — contributions welcome.
