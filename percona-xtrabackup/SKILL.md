---
name: percona-xtrabackup
description: "Hot, non-blocking physical backups of MySQL and Percona Server with Percona XtraBackup (PXB). Use when backing up or restoring MySQL/Percona Server/Percona XtraDB Cluster, choosing between physical and logical backups, setting up full or incremental backups, streaming/compressing/encrypting backups, or scripting backup automation. IMPORTANT — Percona XtraBackup is strictly version-matched: PXB major version must equal the server major version (PXB 8.4 backs up only 8.4, NOT 8.0 or 9.x). A backup is NOT usable until it is `--prepare`d. The `innobackupex` wrapper has been removed; use the `xtrabackup` binary directly."
---

# Percona XtraBackup (PXB)

*Last updated: 2026-05-26*

Percona XtraBackup is a 100% open-source tool for **hot, non-blocking physical backups** of InnoDB, MyRocks, and MyISAM data. It copies data files while the server runs, then replays InnoDB redo/undo logs in a `--prepare` step to produce a consistent snapshot. Unlike `mysqldump` (logical, slow to restore), restore is a file copy; unlike the MySQL clone plugin, PXB supports incrementals, compression, encryption, and streaming. It is what Percona XtraDB Cluster and the Percona Operators use under the hood.

> **Version matching is a hard rule.** PXB major version must equal the server major version. PXB 8.4 backs up MySQL/Percona Server/PXC **8.4 only** — not 8.0, not 9.x. Within 8.4 LTS, any PXB 8.4.x can back up any server 8.4.x (minor versions need not match). Use PXB 8.0 for 8.0 servers. `--no-server-version-check` overrides the guard but risks a corrupt backup — do not use it to cross major versions.

## What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| `innobackupex --backup ...` | Removed in PXB 8.x. Use the `xtrabackup` binary directly — there is no wrapper |
| Using PXB 8.4 to back up an 8.0 (or 9.x) server | Unsupported. Match the major version: PXB 8.0 ↔ server 8.0, 8.4 ↔ 8.4, 9.x ↔ 9.x |
| `--copy-back` straight after `--backup` | You must `--prepare` first. InnoDB will refuse to start on unprepared files |
| Restoring into a populated datadir, or with `mysqld` running | Stop the server and empty the datadir before `--copy-back`/`--move-back` |
| Forgetting to fix ownership after restore | PXB keeps the OS user's ownership; run `chown -R mysql:mysql /var/lib/mysql` before starting `mysqld` |
| `--prepare` with no `--apply-log-only` while more incrementals remain | Once rollback runs, the chain is dead. Use `--apply-log-only` on every step except the **final** incremental |
| Granting `SUPER` to the backup user | Wrong grants. Need `BACKUP_ADMIN, PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT` + `SELECT` on `performance_schema.log_status` (and `keyring_component_status`/`replication_group_members` as applicable) |
| Assuming `--compress` means lz4 | Default is **zstd**. Use `--compress=lz4` for lz4. QuickLZ is gone in 8.4 |
| Expecting incremental MyRocks backups to be small | MyRocks files are copied **in full** every backup — only InnoDB benefits from incrementals |
| `FLUSH TABLES WITH READ LOCK` to "help" the backup | PXB already uses lightweight **backup locks** automatically; don't add a global read lock |

## Full Backup → Prepare → Restore

```bash
# 1. Backup (parallel file copy)
xtrabackup --backup --parallel=4 \
  --user=bkpuser --password='s3cr%T' --target-dir=/data/backups/base

# 2. Prepare — apply redo/undo to make it consistent (MANDATORY)
xtrabackup --prepare --target-dir=/data/backups/base

# 3. Restore — server stopped, datadir empty
systemctl stop mysql
xtrabackup --copy-back --target-dir=/data/backups/base    # or --move-back
chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
```

Backup user grants:
```sql
CREATE USER 'bkpuser'@'localhost' IDENTIFIED BY 's3cr%T';
GRANT BACKUP_ADMIN, PROCESS, RELOAD, LOCK TABLES, REPLICATION CLIENT ON *.* TO 'bkpuser'@'localhost';
GRANT SELECT ON performance_schema.log_status TO 'bkpuser'@'localhost';
```

PXB writes progress to **stderr** (stdout is reserved for streaming) — redirect with `2> backup.log`.

## Incremental Backups

Incrementals copy only pages changed since a base (tracked by LSN in `xtrabackup_checkpoints`).

```bash
xtrabackup --backup --target-dir=/data/backups/base
xtrabackup --backup --target-dir=/data/backups/inc1 --incremental-basedir=/data/backups/base
xtrabackup --backup --target-dir=/data/backups/inc2 --incremental-basedir=/data/backups/inc1
```

Prepare order is critical — `--apply-log-only` on the base and every incremental **except the last**:
```bash
xtrabackup --prepare --apply-log-only --target-dir=/data/backups/base
xtrabackup --prepare --apply-log-only --target-dir=/data/backups/base --incremental-dir=/data/backups/inc1
xtrabackup --prepare               --target-dir=/data/backups/base --incremental-dir=/data/backups/inc2
```
Never prepare the same incremental dir twice (corrupts the backup). **Page tracking** (`--page-tracking`, requires `INSTALL COMPONENT "file://component_mysqlbackup";` and `BACKUP_ADMIN`) makes incrementals faster by reading changed-page IDs instead of scanning; it is incompatible with `--lock-ddl=REDUCED`.

## Compression, Encryption, Streaming

```bash
# Compress (zstd default; level 1=fast..19=small) and decompress
xtrabackup --backup --compress --compress-threads=4 --target-dir=/data/backup
xtrabackup --decompress --remove-original --target-dir=/data/backup

# Encrypt with a key file (preferred over inline --encrypt-key)
xtrabackup --backup --encrypt=AES256 --encrypt-key-file=/etc/pxb.key --target-dir=/data/backup
xtrabackup --decrypt=AES256 --encrypt-key-file=/etc/pxb.key --target-dir=/data/backup   # before --prepare

# Stream to a remote host
xtrabackup --backup --compress --stream | ssh user@host "xbstream -x -C /data/restore"
```
If a backup is both compressed and encrypted: **decrypt first, then decompress, then prepare.**

## DDL Locking & PXC

- `--lock-ddl=ON` (default) blocks DDL for the whole backup; `--lock-ddl=REDUCED` (8.4.0-5+) holds the lock only briefly — big win on large datasets, but incompatible with `--page-tracking`. `--lock-ddl-per-table` is deprecated.
- Backing up a **PXC** node: add `--galera-info` (writes `xtrabackup_galera_info` with the cluster `uuid:seqno`). On restore for a bootstrap node, set `safe_to_bootstrap: 1` in `grastate.dat` (others `0`).

## Partial Backups

Require `innodb_file_per_table=ON`, select with `--tables` / `--databases`. Partial backups are restored by **importing tablespaces**, not `--copy-back`, and cannot be the base for incrementals.

## Sources

- [How XtraBackup works](https://docs.percona.com/percona-xtrabackup/8.4/how-xtrabackup-works.html)
- [Server/backup version comparison](https://docs.percona.com/percona-xtrabackup/8.4/server-backup-version-comparison.html)
- [Connection & privileges](https://docs.percona.com/percona-xtrabackup/8.4/privileges.html)
- [Prepare a full backup](https://docs.percona.com/percona-xtrabackup/8.4/prepare-full-backup.html)
- [Incremental backups](https://docs.percona.com/percona-xtrabackup/8.4/create-incremental-backup.html)
- [Compress & encrypt](https://docs.percona.com/percona-xtrabackup/8.4/create-compressed-backup.html)
- [Reduced-lock backups](https://docs.percona.com/percona-xtrabackup/8.4/reduction-in-locks.html)

*Replace `8.4` with your major version. For anything not covered here, see [docs.percona.com/percona-xtrabackup](https://docs.percona.com/percona-xtrabackup).*
