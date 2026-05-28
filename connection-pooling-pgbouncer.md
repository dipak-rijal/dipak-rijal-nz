# 🐘 Connection Pooling with PgBouncer — A Production Field Guide

**By [Dipak Rijal](https://www.rijal.co.nz)** · Oracle Certified Master · PostgreSQL SME

If your Postgres database is falling over under load, runs out of connections at the worst possible moment, or burns RAM faster than you can provision it — you almost certainly have a connection problem, not a query problem. This is the note I wish every team had read before they cranked `max_connections` to 1000 and called it a day.

---

## Why raising `max_connections` is the wrong fix

Every Postgres connection is a separate operating-system process — not a lightweight thread. Each backend reserves memory for its work area, sits in the process table, and adds to the scheduler's context-switching load. A few hundred mostly-idle connections quietly eat gigabytes of RAM and degrade throughput long before you hit any documented limit.

The instinct under load is to raise `max_connections`. That treats the symptom and worsens the disease: more processes competing for the same CPUs means *less* useful work gets done. Past a point, adding connections reduces total throughput. Postgres does many things brilliantly; managing thousands of short-lived client connections is not one of them.

That's the job of a connection pooler.

---

## What a pooler actually does

A pooler sits between your application and Postgres. The app opens a connection to the pooler — cheap and fast — and the pooler hands it a *reused* backend connection from a small, warm pool. When the app is done, the backend goes back in the pool for the next client.

The result: hundreds or thousands of application clients can be served by a couple of dozen real Postgres backends. **PgBouncer** is the long-standing, battle-tested choice. It's tiny (a single process), it's fast, and it's been holding production together for well over a decade.

---

## The three pool modes — and the one that bites people

PgBouncer can return a server connection to the pool at three different moments. This single setting is the most important decision you'll make.

### Session mode
The server connection is tied to one client for the entire life of that client's connection. Safe and fully transparent — every Postgres feature works — but it gives you the least pooling benefit, because an idle client still holds a backend hostage.

### Transaction mode
The server connection is returned to the pool the instant a transaction ends (`COMMIT` or `ROLLBACK`). This is where the real win lives: a small pool can serve enormous client concurrency, because connections are only held while a transaction is genuinely active. **This is the mode most high-traffic systems should run.**

The catch: anything that relies on *session* state breaks, because consecutive statements from one client may land on different backends. That means session-level `SET`, session advisory locks, `LISTEN`/`NOTIFY`, and `WITH HOLD` cursors don't behave as expected. Design around them.

### Statement mode
The connection is returned after every single statement. Multi-statement transactions are forbidden entirely. Niche — reach for it only when you know exactly why.

---

## The prepared-statement gotcha (and the fix)

Historically, prepared statements and transaction mode were mutually exclusive. A prepared statement lives on one specific backend; in transaction mode your next query might land on a different backend that's never seen it — producing the dreaded `prepared statement does not exist` error.

This is now solved. Since **PgBouncer 1.21**, transaction mode can track prepared statements and transparently re-prepare them on whichever backend serves the request. You enable it by setting `max_prepared_statements` to a non-zero value:

```ini
[pgbouncer]
pool_mode = transaction
max_prepared_statements = 200
```

Two things worth knowing before you rely on it:

- **Upgrade first.** Versions before 1.21 simply do not support this — adding `max_prepared_statements` to an older config will refuse to start. 1.22 later added clean handling of `DISCARD ALL` / `DEALLOCATE ALL`.
- **Driver compatibility.** Most drivers are fine. PHP/PDO is the notable exception — it needs PHP 8.4+ with libpq 17 to play nicely; on older stacks, disable client-side prepared statements instead.

---

## Sizing the pool — smaller than you think

The most common mistake after "too many connections" is "too big a pool." Your pool size should be governed by how many queries your hardware can *actually* run in parallel, not by how many clients you have. A useful starting point for the active pool is roughly the number of CPU cores available to Postgres, adjusted up a little for I/O wait — then load-test and tune.

A representative starting config:

```ini
[pgbouncer]
pool_mode = transaction
max_client_conn = 2000      ; how many app clients can connect to PgBouncer
default_pool_size = 25      ; real backends per database/user pair
reserve_pool_size = 5       ; emergency headroom for spikes
reserve_pool_timeout = 3
server_idle_timeout = 600
```

`max_client_conn` can be large and cheap. `default_pool_size` is the number that hits Postgres — keep it lean. The whole point is the gap between those two numbers.

---

## Monitoring — PgBouncer's admin console

PgBouncer exposes a virtual `pgbouncer` database you connect to with psql. The commands you'll live in:

- `SHOW POOLS;` — active vs waiting clients per pool. **If `cl_waiting` is persistently above zero, your pool is too small** (or queries are too slow).
- `SHOW STATS;` — query and transaction throughput, average query time.
- `SHOW CLIENTS;` / `SHOW SERVERS;` — who's connected and what state they're in.

Watch `cl_waiting` like a hawk. Clients queueing for a free backend is the single clearest signal that something needs tuning — either the pool size or the queries themselves.

---

## A sane rollout checklist

1. Stand up PgBouncer alongside Postgres (same host is fine to start; move it out later).
2. Start in **session mode** to prove connectivity with zero behavioural surprises.
3. Audit the app for session-state dependencies (`SET`, advisory locks, `LISTEN`/`NOTIFY`).
4. Switch to **transaction mode**; enable `max_prepared_statements` if your driver benefits.
5. Set `default_pool_size` lean, then load-test and watch `SHOW POOLS`.
6. Drop your Postgres `max_connections` back to something sane now that the pooler absorbs the churn.

---

## The bottom line

Connection pooling is the highest-leverage, lowest-cost change most Postgres shops are missing. It's not glamorous — but neither is a database that stops accepting connections during your busiest hour.

> *"The boring operational stuff — monitoring bloat, tuning autovacuum, right-sizing pools — that's what keeps production alive."*

---

🔧 **Free tools for engineers** at **[rijal.co.nz](https://www.rijal.co.nz)** — SQL formatter, cron translator, and a dozen dev utilities. No login, no paywall.

⭐ Found this useful? Star the [repo](https://github.com/dipak-rijal/dipak-rijal-nz) — contributions welcome.
