---
name: percona-server-features
description: "Percona Server for MySQL features and enhancements that go beyond stock Oracle MySQL. Use when working with Percona Server, reviewing MySQL code or schema that runs on Percona Server, asking what Percona Server adds over community MySQL, tuning instrumentation/observability, configuring data-at-rest encryption, auditing, the MyRocks engine, threadpool, or backup locking. Also use when deciding whether Percona Server's extra features help a given workload. IMPORTANT — Percona Server is a TRUE drop-in binary replacement for the matching MySQL version (8.0, 8.4 LTS, 9.x): same SQL, same wire protocol, same replication. Do NOT treat it like MariaDB (which is not a drop-in). The value is extra instrumentation, encryption, and operational features bundled in — not syntax differences."
---

# Percona Server for MySQL — Features Worth Knowing

*Last updated: 2026-05-26*

Percona Server for MySQL is an enhanced, fully open-source, **drop-in replacement** for Oracle MySQL. It tracks upstream MySQL release-for-release and stays binary- and protocol-compatible, then adds instrumentation, security, and operational features — several of which are Enterprise-only or absent in community MySQL. AI agents default to generic MySQL advice and rarely suggest these.

For backups (Percona XtraBackup), `pt-*` tools (Percona Toolkit), clustering (PXC), monitoring (PMM), and Kubernetes, see the sibling skills.

> **Version context:** There is **no single default** — Percona Server ships matched to the MySQL release it is built from: **8.0**, **8.4 LTS**, and **9.x Innovation**. Always confirm the running version (`SELECT VERSION();` → e.g. `8.4.8-8`) before relying on a feature. Per-feature tags below are **minimum versions**. Because Percona Server inherits upstream behavior, MySQL 8.4 changes (e.g. `caching_sha2_password` default, removal of `SHOW SLAVE STATUS` / `CHANGE MASTER TO`) apply to Percona Server 8.4 too — see the `mysql-to-percona-server` skill.

## What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| "Percona Server has different SQL syntax / isn't a real drop-in" | It **is** a drop-in for the matching MySQL version — identical SQL, protocol, and replication. Only *extra* features and some defaults differ |
| Treating Percona Server like MariaDB (no `RETURNING`, `SEQUENCE`, GTID incompatibility) | Wrong product. Percona Server == MySQL semantics. MariaDB-isms do not apply |
| Hand-rolling per-table/per-user query stats from the slow log | Use built-in **User Statistics** (`userstat=ON`) — `INFORMATION_SCHEMA.CLIENT_STATISTICS`, `TABLE_STATISTICS`, `INDEX_STATISTICS`, `USER_STATISTICS` |
| `FLUSH TABLES WITH READ LOCK` to take a consistent backup | Use **backup locks**: `LOCK TABLES FOR BACKUP` + `LOCK BINLOG FOR BACKUP` — far less blocking; this is what XtraBackup uses |
| "Install the audit plugin from MySQL Enterprise" | Percona Server bundles the open-source **Audit Log Filter** component (and the older Audit Log plugin) — no Enterprise license |
| "Use the MySQL Enterprise Thread Pool" | Percona Server includes an open-source **threadpool** (`thread_handling=pool-of-threads`) |
| "Keyring Vault needs MySQL Enterprise" | Percona Server ships the **HashiCorp Vault keyring** component in the open-source build |
| Killing stuck idle transactions with a cron of `KILL` | `SET GLOBAL kill_idle_transaction = N;` ends transactions idle longer than N seconds |
| Storing IP/blob columns uncompressed and writing a compression layer | **Compressed columns with dictionaries** for `VARCHAR`/`BLOB`/`JSON` (`COLUMN_FORMAT COMPRESSED`) |

## Instrumentation & Diagnostics

Percona Server roughly doubles the visibility of stock MySQL (≈95 vs ≈65 `INFORMATION_SCHEMA` tables; ≈850 vs ≈430 global counters).

### User Statistics
Per-thread, per-user, per-client, per-table, and per-index counters — none of which exist in community MySQL.
```sql
SET GLOBAL userstat = ON;                       -- enable (off by default)
SELECT * FROM INFORMATION_SCHEMA.TABLE_STATISTICS ORDER BY ROWS_READ DESC LIMIT 10;
SELECT * FROM INFORMATION_SCHEMA.INDEX_STATISTICS WHERE TABLE_NAME = 'orders';
SELECT * FROM INFORMATION_SCHEMA.USER_STATISTICS;
```
Use this to find unused indexes (zero rows in `INDEX_STATISTICS`) and hot tables without parsing logs.

### Extended Slow Query Log
The slow log carries far more per-query detail than upstream — rows examined/sent, query plan flags, InnoDB I/O, temp tables, and more — controlled by `log_slow_verbosity`. Pairs directly with `pt-query-digest` (see `percona-toolkit`).
```sql
SET GLOBAL log_slow_verbosity = 'full';
SET GLOBAL slow_query_log = ON;
```

Also available: enhanced `SHOW ENGINE INNODB STATUS`, undo-segment and temp-table information, and thread-based profiling.

## Security & Encryption

Percona Server brings several encryption capabilities that are Enterprise-only or missing upstream:

- **Data-at-rest encryption** beyond core: binary/relay logs, temporary files, the doublewrite buffer, and enforcement (`table_encryption_privilege_check`, `default_table_encryption`).
- **Keyring components/plugins**: keyring file *and* **HashiCorp Vault** (Enterprise-only in MySQL).
- **Audit Log Filter** (`8.0.34+` component model) — rule-based auditing with JSON/XML output, compression, and encryption. The older Audit Log plugin is still available; prefer the filter component for new setups.
- **Data Masking** component — `mask_inner()`, `gen_rnd_email()`, etc., open-source (Enterprise-only upstream).
- **PAM authentication** plugin (Enterprise-only upstream).

```sql
-- discover unused indexes, then check encryption coverage
SELECT TABLE_SCHEMA, TABLE_NAME, CREATE_OPTIONS
FROM INFORMATION_SCHEMA.TABLES WHERE CREATE_OPTIONS LIKE '%ENCRYPTION%';
```

## Storage Engines

- **MyRocks** (`8.0+`, `8.4+`) — RocksDB-based LSM engine bundled with Percona Server, optimized for write-heavy workloads and high compression. Install the package and enable it; it is not loaded by default:
  ```bash
  sudo apt install percona-server-rocksdb      # or the yum equivalent
  ps-admin --enable-rocksdb                     # registers the plugin + tablespaces
  ```
  ```sql
  CREATE TABLE events (id BIGINT PRIMARY KEY, payload JSON) ENGINE=ROCKSDB;
  ```
  MyRocks has real limitations (gap locks, foreign keys, mixing engines in a transaction) — see [MyRocks limitations](https://docs.percona.com/percona-server/8.4/myrocks-limitations.html) before adopting.
- **Improved MEMORY engine** and InnoDB enhancements (configurable fast index creation, full-text search improvements, corrupt-table handling via `innodb_corrupt_table_action`).

## Operational Features

| Feature | What it gives you | Min version |
|---|---|---|
| **Backup locks** — `LOCK TABLES FOR BACKUP`, `LOCK BINLOG FOR BACKUP` | Lightweight locking for consistent backups without blocking reads/most writes | 8.0+ |
| **Kill idle transactions** — `kill_idle_transaction` | Auto-end transactions idle past a timeout, freeing locks/undo | 8.0+ |
| **Threadpool** — `thread_handling=pool-of-threads` | Scales connection counts without thread-per-connection overhead | 8.0+ |
| **Extended `SHOW GRANTS`** | Shows roles and all grants in one statement | 8.0+ |
| **Compressed columns with dictionaries** — `COLUMN_FORMAT COMPRESSED` | Per-column compression for `VARCHAR`/`BLOB`/`JSON` | 8.0+ |
| **Enforce storage engine** — `enforce_storage_engine` | Reject tables created with the wrong engine (DBaaS guardrail) | 8.0+ |
| **Improved `START TRANSACTION WITH CONSISTENT SNAPSHOT`** | Consistent snapshot incl. binlog position for backups | 8.0+ |

```sql
-- backup-safe locking (what XtraBackup does internally)
LOCK TABLES FOR BACKUP;
LOCK BINLOG FOR BACKUP;
-- ... copy / snapshot ...
UNLOCK BINLOG;
UNLOCK TABLES;
```

## Percona Server Pro

Percona also offers **Percona Server Pro** builds (extra packaging/support features) on top of the open-source server. The core enhancements above are in the standard open-source build; do not assume a feature requires Pro unless the docs say so.

## Sources

- [Percona Server feature comparison](https://docs.percona.com/percona-server/8.4/feature-comparison.html)
- [User Statistics](https://docs.percona.com/percona-server/8.4/user-stats.html)
- [Extended slow query log](https://docs.percona.com/percona-server/8.4/slow-extended.html)
- [Backup locks](https://docs.percona.com/percona-server/8.4/backup-locks.html)
- [Data-at-rest encryption](https://docs.percona.com/percona-server/8.4/data-at-rest-encryption.html)
- [Audit Log Filter overview](https://docs.percona.com/percona-server/8.4/audit-log-filter-overview.html)
- [MyRocks](https://docs.percona.com/percona-server/8.4/myrocks-index.html)
- [Threadpool](https://docs.percona.com/percona-server/8.4/threadpool.html)

*Replace `8.4` in the URLs with your server's major version. For anything not covered here, see the official docs at [docs.percona.com/percona-server](https://docs.percona.com/percona-server).*
