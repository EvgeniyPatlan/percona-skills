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
| `xtrabackup --backup --stream \| xbcloud put …` | Use `--stream=xbstream` — bare `--stream` won't pipe to xbcloud; and check `${PIPESTATUS[0]}`, not `$?` |
| Reading `xtrabackup_slave_info` for PITR on the backup's own host | That file holds the *source's* coordinates (for seeding a replica). For PITR use `xtrabackup_binlog_info` |
| `--slave-info` alone when backing up a replica | Pair it with `--safe-slave-backup`, or the backup can be inconsistent if temp tables are open |
| `CHANGE MASTER TO` / `START SLAVE` when rebuilding a replica | 8.4 removed them; use `CHANGE REPLICATION SOURCE TO` / `START REPLICA` (what `xtrabackup_slave_info` already writes) |

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

## Cloud Backups (xbcloud)

Stream a backup straight to S3 / GCS / Azure / Swift / MinIO with no local staging — the pattern is `xtrabackup --backup --stream=xbstream | xbcloud put`. Note `--stream=xbstream` (not bare `--stream`) is required.

```bash
# Backup → S3
xtrabackup --backup --stream=xbstream --parallel=4 2>backup.log | \
  xbcloud put --storage=s3 --s3-bucket=mysql_backups --parallel=10 \
    --s3-endpoint='s3.amazonaws.com' \
    --s3-access-key="$AWS_KEY" --s3-secret-key="$AWS_SECRET" \
    "$(date -I)-full"

# Restore from S3
xbcloud get s3://mysql_backups/2026-05-27-full --parallel=10 2>dl.log | \
  xbstream -x -C /var/lib/mysql-restore --parallel=8

xbcloud delete s3://mysql_backups/2026-05-27-full
```
Put credentials in a `[xbcloud]` group in `my.cnf` to keep them off the command line. In a pipe, check `${PIPESTATUS[0]}` for xtrabackup's exit code — `$?` only reflects xbcloud.

## Point-in-Time Recovery

Restore a physical backup, then replay binary logs from the position the backup recorded in `xtrabackup_binlog_info`.
```bash
xtrabackup --copy-back --target-dir=/backups/base && chown -R mysql:mysql /var/lib/mysql
systemctl start mysql
cat /backups/base/xtrabackup_binlog_info          # e.g. mysql-bin.000003   57
mysqlbinlog /var/lib/mysql/mysql-bin.000003 /var/lib/mysql/mysql-bin.000004 \
  --start-position=57 --stop-datetime="2026-05-27 14:00:00" | mysql -u root -p
```

## Rebuild a Replica From a Backup

The most common production use. Back up the source (or an existing replica with `--slave-info --safe-slave-backup`), prepare, copy to the new node, then point replication at the recorded coordinates.
```bash
# GTID-based (preferred): xtrabackup_binlog_info line 3 holds the GTID set
mysql -e "RESET BINARY LOGS AND GTIDS;
          SET GLOBAL gtid_purged='<gtid-set-from-xtrabackup_binlog_info>';
          CHANGE REPLICATION SOURCE TO SOURCE_HOST='src', SOURCE_USER='repl',
            SOURCE_PASSWORD='***', SOURCE_AUTO_POSITION=1;
          START REPLICA;"
```
`--slave-info` writes a ready-made `CHANGE REPLICATION SOURCE TO` into `xtrabackup_slave_info`. Use `xtrabackup_binlog_info` for the backup's own server; use `xtrabackup_slave_info` when seeding from a replica.

## Single-Table Restore (export / import tablespace)

For partial backups or recovering one table. Requires `innodb_file_per_table=ON` and the same major version on both ends.
```bash
xtrabackup --prepare --export --target-dir=/backups/base      # makes .cfg (+ .cfp if encrypted)
# on target: recreate the table DDL exactly, then:
mysql db -e "ALTER TABLE t DISCARD TABLESPACE;"
cp /backups/base/db/t.{ibd,cfg} /var/lib/mysql/db/ && chown mysql:mysql /var/lib/mysql/db/t.*
mysql db -e "ALTER TABLE t IMPORT TABLESPACE;"
```

## Other Useful Options

- `--throttle=N` — cap I/O to N×10 MB/s on shared hosts (too low risks redo-log wraparound; pair with `--register-redo-log-consumer` on write-heavy replicas).
- `--safe-slave-backup --slave-info` — the safe combo when backing up a replica.
- `--history=<name>` — record each run in `PERCONA_SCHEMA.XTRABACKUP_HISTORY`; base incrementals on it with `--incremental-history-name`.
- Partial backups: `innodb_file_per_table=ON`, select with `--tables` / `--databases`; restored by import (above), never `--copy-back`, and cannot be an incremental base.
- Large `--prepare`: `--use-memory=2G` is far faster than the 128 MB default (`--use-free-memory-pct=50` also works, but only if the backup was taken with `--estimate-memory=ON`).

## Sources

- [How XtraBackup works](https://docs.percona.com/percona-xtrabackup/8.4/how-xtrabackup-works.html)
- [Server/backup version comparison](https://docs.percona.com/percona-xtrabackup/8.4/server-backup-version-comparison.html)
- [Connection & privileges](https://docs.percona.com/percona-xtrabackup/8.4/privileges.html)
- [Prepare a full backup](https://docs.percona.com/percona-xtrabackup/8.4/prepare-full-backup.html)
- [Incremental backups](https://docs.percona.com/percona-xtrabackup/8.4/create-incremental-backup.html)
- [Compress & encrypt](https://docs.percona.com/percona-xtrabackup/8.4/create-compressed-backup.html)
- [Reduced-lock backups](https://docs.percona.com/percona-xtrabackup/8.4/reduction-in-locks.html)
- [Backup to cloud storage (xbcloud)](https://docs.percona.com/percona-xtrabackup/8.4/xbcloud-binary-overview.html)
- [Point-in-time recovery](https://docs.percona.com/percona-xtrabackup/8.4/point-in-time-recovery.html)
- [Restore individual tables](https://docs.percona.com/percona-xtrabackup/8.4/restore-individual-tables.html)

*Replace `8.4` with your major version. For anything not covered here, see [docs.percona.com/percona-xtrabackup](https://docs.percona.com/percona-xtrabackup/).*
