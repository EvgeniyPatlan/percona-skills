---
name: percona-toolkit
description: "The Percona Toolkit pt-* command-line tools for MySQL, Percona Server, and MariaDB administration. Use when performing an online schema change (ALTER without locking), analyzing slow queries, archiving/purging rows, checking or fixing replication consistency, killing runaway queries, measuring real replication lag, or finding duplicate/unused indexes. IMPORTANT — most data-changing tools require an explicit --execute (or have no safety flag at all, like pt-archiver), so always --dry-run / --print first. Prefer native online DDL (ALGORITHM=INSTANT/INPLACE) over pt-online-schema-change when the operation qualifies."
---

# Percona Toolkit (`pt-*`)

*Last updated: 2026-05-26*

Percona Toolkit is a collection of mature, tested command-line tools for MySQL, Percona Server, and MariaDB. Install the `percona-toolkit` package (`percona-release enable pt release` first). Tools are mostly server-version independent and connect via DSN syntax (`h=host,P=port,D=db,t=table,u=user,p=pass`). AI agents reach for hand-written scripts when one of these tools is the safer, proven answer.

> **Safety first.** `pt-online-schema-change` and `pt-table-sync` require `--execute`; without it they only check/print. `pt-archiver` has **no** execute guard — running it acts immediately, so always `--dry-run` first. `pt-kill` needs `--kill` (use `--print` to preview).

## What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| `pt-online-schema-change --alter "..." D=db,t=t` and expecting it to run | It only runs checks without `--execute`. Add `--execute` explicitly (mutually exclusive with `--dry-run`) |
| Running `pt-online-schema-change` for `ADD COLUMN ... DEFAULT` on MySQL 8.0.12+ | That is an `ALGORITHM=INSTANT` change — milliseconds, no copy. Use native DDL; pt-osc is needless overhead here |
| pt-osc on a table with no PRIMARY KEY / UNIQUE index | Refused — the DELETE trigger needs one (unless the ALTER itself creates it) |
| pt-osc on an FK-referenced table with no FK option | Must pass `--alter-foreign-keys-method` (`auto`/`rebuild_constraints`/`drop_swap`). `drop_swap` is blocked on MySQL 8.0.0–8.0.14 (bug 89441) |
| `--alter "DROP FOREIGN KEY fk_x"` in pt-osc | Use `DROP FOREIGN KEY _fk_x` — pt-osc prefixes FK names with `_` on the new table |
| Adding a UNIQUE index via pt-osc on possibly-duplicate data | pt-osc copies with `INSERT IGNORE` — duplicates are silently dropped. It warns by default; don't disable the check |
| `pt-table-sync --execute h=src,... dst` in a source↔source topology | Writes directly to `dst` and replicates back, corrupting `src`. Use `--sync-to-source` so changes flow through replication |
| Trusting `Seconds_Behind_Source` for lag in automation | Unreliable. Use `pt-heartbeat --monitor`, which measures actual replicated timestamps |
| Running `pt-archiver` to "just test it" | No dry-run default — it deletes/copies immediately. Always `--dry-run` first and verify the index plan |
| `pt-archiver … --limit 1000 --commit-each` on millions of rows | Add `--bulk-delete`; otherwise it's one DELETE *per row*. Default `--txn-size=1` also needs raising |
| `pt-kill --busy-time 60 --kill` expecting all matches to die | Default `--victims oldest` kills only the oldest per class. Use `--victims all`; `--kill-query` cancels the query but keeps the connection |
| Raising pt-osc `--max-load` without touching `--critical-load` | `--critical-load` (default 50) aborts the run and orphans the shadow table — keep it above your `--max-load` |
| `p=secret` in a DSN on the command line | Use `F=/root/.my.cnf` or `--ask-pass`; CLI passwords leak via `ps` and shell history |

## High-Value Tools

### pt-online-schema-change — online ALTER
Builds a shadow table with the new schema, copies rows in chunks, uses triggers to capture concurrent writes, then atomically `RENAME`s it in. Throttles on replica lag.
```bash
pt-online-schema-change \
  --alter "ADD COLUMN status TINYINT NOT NULL DEFAULT 0" \
  --max-lag=1 --execute D=shop,t=orders
```
Defaults: `--chunk-time=0.5s`, `--chunk-size=1000`, `--max-lag=1`. It sets `innodb_lock_wait_timeout=1` so it (not your app) is the lock victim. On MySQL < 5.7.2 it cannot run on a table that already has triggers; on 5.7.2+ use `--preserve-triggers`. **Load throttling:** `--max-load` (default `Threads_running=25`) *pauses* the copy; `--critical-load` (default `=50`) *aborts* and leaves the shadow table behind. `--chunk-size-limit=4.0` aborts a chunk whose EXPLAIN estimates >4× the target rows (a bad-index guard). On managed DBs (RDS/Aurora) where replicas aren't in `PROCESSLIST`, use `--recursion-method=dsn=D=percona,t=dsns` or `=none`.

**pt-osc vs native DDL vs gh-ost.** Prefer native `ALGORITHM=INSTANT` (8.0.12+: add column, set default, rename column) or `INPLACE` (add/drop index) — they avoid the copy entirely. Use pt-osc (or gh-ost) only when the change would otherwise force `ALGORITHM=COPY` on a large hot table, or when you need replica-lag throttling native DDL lacks. pt-osc uses triggers (no setup, works on PXC with `wsrep_OSU_method=TOI`); gh-ost reads the binlog instead (no trigger overhead, needs ROW binlog).

### pt-query-digest — slow query analysis
Groups queries by fingerprint and ranks by total time. Reads slow/general/binary logs, processlist, or tcpdump.
```bash
pt-query-digest /var/log/mysql/slow.log
pt-query-digest --processlist h=db1            # live sampling
```
Pairs directly with Percona Server's extended slow log (`log_slow_verbosity='full'`). For continuous, live analysis, PMM Query Analytics is the always-on equivalent (see `percona-monitoring-and-management`).

### pt-archiver — low-impact purge/archive
Nibbles rows in forward index scans to a `--dest` table, a `--file`, or `--purge` (delete). No execute guard — `--dry-run` first.
```bash
pt-archiver --source h=db,D=shop,t=events --purge \
  --where "created_at < '2025-01-01'" --limit 1000 --bulk-delete --sleep 1
```
Defaults bite: `--txn-size=1` and `--limit=1` mean one row per transaction/SELECT — raise both. For big purges add `--bulk-delete` (one ranged DELETE per chunk instead of per-row); for big moves add `--bulk-insert` (LOAD DATA on the dest).

### pt-table-checksum + pt-table-sync — replication consistency
`pt-table-checksum` runs `REPLACE...SELECT` checksums on the source that replicate down and reveal drift (writes to `percona.checksums`, non-zero exit on diff). `pt-table-sync` repairs it — always via `--sync-to-source` so fixes flow through replication, never written straight to a replica.
```bash
pt-table-checksum --databases shop h=source
pt-table-sync --execute --sync-to-source --replicate percona.checksums h=replica1
```

### Others
- **pt-heartbeat** — true replication lag via a heartbeat row (`--update` on source, `--monitor` on replica); needs synced clocks.
- **pt-kill** — kill connections matching criteria (`--busy-time`, `--match-info`); `--print` to preview.
- **pt-stalk** / **pt-pmp** — capture diagnostics when a trigger (e.g. `Threads_running`) fires; `pt-pmp` aggregates stack traces (use eu-stack, not GDB, in production). Never pass `--password` on the CLI (visible in `ps`).
- **pt-duplicate-key-checker** — redundant indexes/FKs → `DROP INDEX` suggestions.
- **pt-index-usage** — replays a slow log to find never-used indexes.
- **pt-show-grants** — canonical, diff-able grant dump (great for version control).
- **pt-mysql-summary** / **pt-summary** — read-only server / host snapshots.
- **pt-upgrade** — replay queries against two servers and diff results/errors/row-counts before a version upgrade (`pt-upgrade h=old h=new slow.log`); non-zero exit on any difference, so it scripts well in CI.
- **pt-variable-advisor** — flag risky `SHOW VARIABLES` settings.
- **pt-config-diff** — diff `SHOW VARIABLES` between two servers, or a `my.cnf` vs a live server (`pt-config-diff /etc/my.cnf h=prod`).
- **pt-deadlock-logger** / **pt-fk-error-logger** — parse `SHOW ENGINE INNODB STATUS` for deadlocks / FK errors; log to stdout or a table.
- **pt-replica-restart** (was pt-slave-restart) — auto-restart replication past *named* transient errors (`--error-numbers=1062`); always follow with `pt-table-checksum` since skipping diverges data.
- **pt-find** — `find(1)` for tables (`pt-find --engine MyISAM --exec "ALTER TABLE %D.%N ENGINE=InnoDB"`).
- **pt-fingerprint** / **pt-visual-explain** — normalize a query to its fingerprint; turn `EXPLAIN` into a readable tree.

## DSNs & Credentials

Tools take a DSN: `h=host,P=3306,u=user,p=pass,D=db,t=table,S=/sock,F=/path/my.cnf`. **Never put `p=secret` on the command line** (visible in `ps`, saved to shell history) — use `F=/root/.my.cnf` (reads the `[client]` stanza) or `--ask-pass`. Escape commas in passwords as `\,`.
```ini
# /root/.my.cnf  →  then DSNs shrink to just D=shop,t=orders
[client]
user=ptuser
password=secret
host=db1
```

## Sources

- [pt-online-schema-change](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
- [pt-query-digest](https://docs.percona.com/percona-toolkit/pt-query-digest.html)
- [pt-archiver](https://docs.percona.com/percona-toolkit/pt-archiver.html)
- [pt-table-checksum](https://docs.percona.com/percona-toolkit/pt-table-checksum.html) · [pt-table-sync](https://docs.percona.com/percona-toolkit/pt-table-sync.html)
- [pt-heartbeat](https://docs.percona.com/percona-toolkit/pt-heartbeat.html) · [pt-kill](https://docs.percona.com/percona-toolkit/pt-kill.html)
- [pt-upgrade](https://docs.percona.com/percona-toolkit/pt-upgrade.html) · [pt-stalk](https://docs.percona.com/percona-toolkit/pt-stalk.html) · [pt-config-diff](https://docs.percona.com/percona-toolkit/pt-config-diff.html)
- [Percona Toolkit docs (each tool's page documents its DSN options)](https://docs.percona.com/percona-toolkit/)

*Each tool also has full docs via `man pt-<tool>` and `pt-<tool> --help`. For anything not covered here, see [docs.percona.com/percona-toolkit](https://docs.percona.com/percona-toolkit/).*
