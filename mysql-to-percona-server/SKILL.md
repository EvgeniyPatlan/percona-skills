---
name: mysql-to-percona-server
description: "Adopting or migrating to Percona Server for MySQL from community (Oracle) MySQL or from MariaDB. Use when migrating an application or server to Percona Server, asking about Percona Server compatibility with MySQL, swapping community MySQL for Percona Server, or upgrading across MySQL major versions (8.0 to 8.4). IMPORTANT — Percona Server is a TRUE binary drop-in replacement for the matching MySQL version: same SQL, protocol, replication, and on-disk format. Migrating from community MySQL of the SAME major version is a package swap (no dump/reload). This is the OPPOSITE of MariaDB, which is NOT a drop-in. The real work is the upstream MySQL 8.4 breaking changes (removed SHOW SLAVE STATUS, caching_sha2_password default, etc.), which Percona Server inherits."
---

# Migrating to Percona Server for MySQL

*Last updated: 2026-05-26*

Percona Server is a fully compatible, enhanced, open-source **drop-in replacement** for community MySQL at the same major version — same SQL dialect, wire protocol, replication format, and InnoDB on-disk layout. Moving from community MySQL 8.4 to Percona Server 8.4 is operationally: stop the server, swap packages, start. No export, no schema rewrite, no app changes.

This is the **opposite** of MariaDB, which diverged after MySQL 5.6 and is *not* a drop-in: moving off MariaDB always needs a logical dump/load. Don't carry MariaDB migration assumptions (GTID incompatibility, `RETURNING`, sequences, auth differences) into a Percona Server migration — they don't apply. Percona Server only *adds* features (see `percona-server-features`); it removes/redefines nothing community MySQL provides.

> **Versions:** Percona Server ships matched to **8.0**, **8.4 LTS**, and **9.x Innovation**. Same-major migrations are package swaps; cross-major (8.0→8.4) is a real upgrade governed by upstream MySQL breaking changes. Verify after with `SELECT VERSION();` → e.g. `8.4.x-...-Percona Server`.

## What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| Treating Percona Server like MariaDB — "dump and reload, rewrite SQL" | It's a true binary drop-in for the same major version. Package swap; datadir is compatible; no dump, no SQL rewrite |
| Moving same-major community MySQL via `mysqldump` | Unnecessary. Keep the datadir and swap the packages in place |
| `SHOW SLAVE STATUS` / `CHANGE MASTER TO` / `START SLAVE` in runbooks on 8.4 | Removed in 8.4. Use `SHOW REPLICA STATUS`, `CHANGE REPLICATION SOURCE TO`, `START REPLICA` |
| Assuming `mysql_native_password` works by default on 8.4 | Disabled by default; needs `mysql_native_password=ON` in `my.cnf`. Default auth is `caching_sha2_password`. Removed entirely in 9.x |
| `default_authentication_plugin=...` in `my.cnf` on 8.4 | Variable removed. Use `authentication_policy` |
| `expire_logs_days=N` on 8.4 | Removed. Use `binlog_expire_logs_seconds` |
| Carrying an 8.0 `my.cnf` verbatim onto 8.4 | Removed variables prevent startup; several InnoDB defaults changed. Audit against the removed-items list first |
| Pointing Percona Server at a MariaDB datadir | Not supported (not binary-compatible). Use logical dump/load into a fresh instance |
| Backing up an 8.4 server with XtraBackup 8.0 | XtraBackup major must match the server major (see `percona-xtrabackup`) |
| Relying on distro MySQL packages | Those are community MySQL. Set up the Percona repo (`percona-release setup ps-84-lts`) and install `percona-server-server` |
| Assuming Threadpool / Vault keyring need a paid tier | Both are free in Percona Server community (Enterprise-only in MySQL) |

## Migration Paths

### Community MySQL X.Y → Percona Server X.Y (same major = package swap)
Datadir is fully compatible; no `mysqldump`.
```bash
# 1. Back up first; save my.cnf
systemctl stop mysql
# 2. Add the Percona repo
curl -O https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo apt install -y ./percona-release_latest.generic_all.deb
sudo percona-release setup ps-84-lts        # or 'ps80' for 8.0
sudo apt update
# 3. Install Percona Server (replaces community packages)
sudo apt install -y percona-server-server
# 4. Start — mysqld runs any needed upgrade step automatically
systemctl start mysql
mysql -e "SELECT VERSION();"
```
RPM is analogous (`percona-release-latest.noarch.rpm`, `percona-release setup ps-84-lts`, `yum install percona-server-server`).

### MariaDB → Percona Server (NOT a drop-in)
MariaDB is not binary-compatible. Logical migration only: `mysqldump`/`mydumper` from MariaDB → load into a fresh Percona Server instance → validate. Never point Percona Server at a MariaDB datadir.

### Percona Server 8.0 → 8.4 (cross-major upgrade)
A real version upgrade — all MySQL 8.4 breaking changes apply. Run the MySQL Shell upgrade checker (`util.checkForServerUpgrade()`), clean `my.cnf` of removed variables, update replication scripts to the SOURCE/REPLICA syntax, upgrade XtraBackup to 8.4, then swap packages and restart. There is no supported in-place downgrade from 8.4 to 8.0 — roll back via a pre-upgrade backup.

## MySQL 8.4 Breaking Changes Percona Server Inherits

- **Auth:** `mysql_native_password` disabled by default (re-enable with `mysql_native_password=ON`; gone in 9.x); `caching_sha2_password` is the default — clients/connectors must support it + TLS. `default_authentication_plugin` removed → `authentication_policy`.
- **Replication syntax removed:** `CHANGE MASTER TO`→`CHANGE REPLICATION SOURCE TO`, `START/STOP SLAVE`→`START/STOP REPLICA`, `SHOW SLAVE STATUS`→`SHOW REPLICA STATUS`, `SHOW MASTER STATUS`→`SHOW BINARY LOG STATUS`, `RESET MASTER`→`RESET BINARY LOGS AND GTIDS`. (The `REPLICATION SLAVE` privilege name is unchanged — don't rename it.)
- **Removed variables:** `expire_logs_days`→`binlog_expire_logs_seconds`; `binlog_transaction_dependency_tracking` (now internal, WRITESET only); `avoid_temporal_upgrade`, `character-set-client-handshake`, `have_openssl`/`have_ssl`, all `innodb_api_*` (memcached gone).
- **Removed function:** `WAIT_UNTIL_SQL_THREAD_AFTER_GTIDS()`→`WAIT_FOR_EXECUTED_GTID_SET()`.
- **New reserved keywords:** `MANUAL`, `PARALLEL`, `QUALIFY`, `TABLESAMPLE` — quote or rename identifiers.
- **Other:** `AUTO_INCREMENT` no longer allowed on `FLOAT`/`DOUBLE`. Several InnoDB defaults changed (e.g. `innodb_adaptive_hash_index` OFF, `innodb_change_buffering` none, `innodb_io_capacity` 10000, `innodb_flush_method` O_DIRECT) — re-evaluate, don't copy an 8.0 config. A spatial-index corruption bug affects 8.4.0–8.4.3; go to 8.4.4+.

## What You Unlock

By switching you gain (all free, open source): User Statistics, the extended slow query log, Threadpool, extra encryption (binlog/relay/temp files, Vault keyring), the Audit Log Filter, backup locks, MyRocks — plus the ecosystem: Percona XtraBackup, Percona Toolkit, PMM, and PXC. See the `percona-server-features` skill and the sibling tool skills.

## Sources

- [Breaking changes (8.4)](https://docs.percona.com/percona-server/8.4/8.4-breaking-changes.html)
- [Compatibility & removed items (8.4)](https://docs.percona.com/percona-server/8.4/8.4-compatibility-and-removed-items.html)
- [Defaults & tuning (8.4)](https://docs.percona.com/percona-server/8.4/8.4-defaults-and-tuning.html)
- [Authentication methods](https://docs.percona.com/percona-server/8.4/authentication-methods.html)
- [Upgrade overview](https://docs.percona.com/percona-server/8.4/upgrade.html) · [APT repo](https://docs.percona.com/percona-server/8.4/apt-repo.html)

*Replace `8.4` with your major version. For anything not covered here, see [docs.percona.com/percona-server](https://docs.percona.com/percona-server).*
