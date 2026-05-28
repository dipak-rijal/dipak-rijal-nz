# 🐘 Database Engineering — Field Notes

**By [Dipak Rijal](https://www.rijal.co.nz)** · Oracle Certified Master · PostgreSQL SME · 15+ years in production

A curated collection of real-world database engineering knowledge — the stuff I wish someone handed me on day one.

---

## 📂 What's inside

### PostgreSQL
- [Bloat detection & vacuum tuning](postgres/bloat-and-vacuum.md) — why your queries slow down over time and how to fix it
- [Autovacuum configuration guide](postgres/autovacuum-tuning.md) — production-ready settings, not defaults
- [Logical replication troubleshooting](postgres/logical-replication.md) — slot bloat, publication drift, decoding lag
- [Query performance checklist](postgres/query-performance.md) — systematic approach to tuning
- [Connection pooling with PgBouncer](https://github.com/dipak-rijal/dipak-rijal-nz/blob/main/connection-pooling-pgbouncer.md) — stop raising max_connections; pool instead

### Oracle
- [Migration playbook: 12c → 19c](oracle/migration-12c-to-19c.md) — lessons from migrating 12 production databases
- [Data Guard setup & monitoring](oracle/dataguard.md) — DR architecture that actually works
- [GoldenGate replication guide](oracle/goldengate.md) — achieving sub-minute replication lag
- [RAC troubleshooting](oracle/rac-troubleshooting.md) — common issues and fixes

### SQL Server
- [Always On AG operations](sqlserver/always-on-ag.md) — setup, failover, monitoring
- [Performance tuning basics](sqlserver/performance-tuning.md) — profiling and query optimisation

### Cloud & Architecture
- [HA/DR architecture patterns](cloud/ha-dr-patterns.md) — Multi-AZ, Multi-Region, RPO/RTO design
- [Cloud migration strategy](cloud/migration-strategy.md) — planning near-zero-downtime migrations
- [FinOps for databases](cloud/finops-databases.md) — right-sizing, reserved instances, storage optimisation
- [Terraform for database infrastructure](cloud/terraform-databases.md) — IaC patterns for RDS, Aurora

### Scripts & Tools
- [Bash utilities for DBAs](scripts/) — monitoring, alerting, backup validation
- [Python automation scripts](scripts/python/) — operational runbooks, data extraction

---

## 🔧 Free tools

I built **[rijal.co.nz](https://www.rijal.co.nz)** — a free resource hub for database and cloud engineers:

- SQL formatter & JSON beautifier
- Cron expression translator
- 12 dev tools (UUID, hash, Base64, regex, password gen)
- Live AWS news + service health dashboard
- Command palette (Ctrl+K) for power users

No login. No paywall.

---

## 🏆 Background

| | |
|---|---|
| 🥇 Oracle Certified Master | Highest Oracle certification globally |
| 🐘 PostgreSQL SME | Recognised expertise in RDS architecture & optimisation |
| ☁️ Oracle Cloud Architect Professional | Cloud platform design |
| 🎓 MIT (Honours) | Auckland University of Technology, NZ |

**15+ years** across India, Switzerland, Thailand, Malaysia and New Zealand — banking, semiconductor manufacturing, government platforms, and enterprise cloud.

---

## 🤝 Connect

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/dipak-rijal)
[![Website](https://img.shields.io/badge/rijal.co.nz-336791?style=flat-square&logo=postgresql&logoColor=white)](https://www.rijal.co.nz)

---

> *"The boring operational stuff — monitoring bloat, tuning autovacuum, reviewing query plans — that's what keeps production alive."*

⭐ Star this repo if you find it useful. Contributions welcome.
