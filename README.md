# Percona Agent Skills

Short **SKILL.md** briefings for AI coding agents — so they get right what they often get wrong about **Percona Server for MySQL** and the wider Percona ecosystem (XtraBackup, Toolkit, XtraDB Cluster, PMM, and the Kubernetes Operators). Skills install on the **AI agent**, not on your database server.

## What is a skill?

A skill is a curated markdown file the agent reads when a topic matches. Think of it as a briefing note: Percona-specific guidance, common LLM mistakes, and pointers to [docs.percona.com](https://docs.percona.com). Skills work with Claude Code, GitHub Copilot, Cursor, OpenAI Codex, and other agent tools.

Skills use wrong/right pairs, version annotations, and links to official documentation so agents can tailor advice to the Percona version you run.

## Why Percona-specific skills?

Percona Server for MySQL is a **true drop-in replacement** for the matching Oracle MySQL release — so agents that know MySQL are already 95% correct on syntax. The gap is everything Percona *adds*: extra instrumentation, encryption, and the operational tooling agents don't know exists, plus a few hard rules they routinely miss (XtraBackup version-matching, PXC's primary-key requirement, MySQL 8.4 removed syntax). These skills close that gap.

## Skills

### [percona-server-features](percona-server-features/SKILL.md)
What Percona Server adds over stock MySQL: User Statistics, the extended slow query log, threadpool, extra encryption and the Vault keyring, the Audit Log Filter, MyRocks, backup locks, and kill-idle-transaction — the things agents rarely suggest unprompted.

### [mysql-to-percona-server](mysql-to-percona-server/SKILL.md)
Adopting or migrating to Percona Server from community MySQL or MariaDB. Why it's a true drop-in (unlike MariaDB), the same-major package swap, and the upstream MySQL 8.4 breaking changes (removed `SHOW SLAVE STATUS`, `caching_sha2_password` default) it inherits.

### [percona-xtrabackup](percona-xtrabackup/SKILL.md)
Hot, non-blocking physical backups. Strict version matching (PXB major must equal server major), the mandatory `--prepare` step, incrementals, compression, encryption, streaming, and PXC backups.

### [percona-toolkit](percona-toolkit/SKILL.md)
The `pt-*` tools: `pt-online-schema-change` (vs native INSTANT DDL), `pt-query-digest`, `pt-archiver`, `pt-table-checksum`/`pt-table-sync`, `pt-stalk`, `pt-kill`, `pt-heartbeat` — and the safety flags agents skip.

### [percona-xtradb-cluster](percona-xtradb-cluster/SKILL.md)
Synchronous multi-master HA with Galera. Primary-key requirement, InnoDB-only replication, certification conflicts at COMMIT, SST/IST, quorum, `pxc_strict_mode`, and TOI-vs-RSU schema changes.

### [percona-query-optimization](percona-query-optimization/SKILL.md)
Diagnosing slow queries and designing indexes using Percona Server's extra instrumentation — `userstat`, the extended slow log, `pt-query-digest`, PMM QAN — plus EXPLAIN ANALYZE, histograms, and invisible indexes.

### [percona-monitoring-and-management](percona-monitoring-and-management/SKILL.md)
Open-source database observability (PMM): client/server architecture, install and registration, and Query Analytics — including the slow-log-vs-Performance-Schema choice and PMM 2 vs 3.

### [percona-operator-for-mysql](percona-operator-for-mysql/SKILL.md)
Running MySQL on Kubernetes. The two distinct operators (PXC-based vs Percona-Server-based), choosing between them, the cluster Custom Resource, scaling, and backups to S3/GCS/Azure.

## How to install

### Interactive install with Node.js

Use [npx skills](https://github.com/vercel-labs/skills) for an interactive install — choose which Percona skills to install and which agents on your system (Claude, Cursor, Codex, etc.):

```
npx skills add EvgeniyPatlan/percona-skills
```

Or install one skill:

```
npx skills add EvgeniyPatlan/percona-skills/<skill-name>
```

Replace `<skill-name>` with e.g. `percona-xtrabackup` or `mysql-to-percona-server`.

### Installing without Node.js

Download `SKILL.md` into the agent's skill folder, for example:

```bash
mkdir -p ~/.claude/skills/<skill-name>
curl -o ~/.claude/skills/<skill-name>/SKILL.md \
  https://raw.githubusercontent.com/EvgeniyPatlan/percona-skills/main/<skill-name>/SKILL.md
```

- Claude Code and Claude Desktop: `~/.claude/skills/<skill-name>/SKILL.md`
- OpenAI Codex: `~/.agents/skills/<skill-name>/SKILL.md`

## Using Percona skills

Once installed, skills activate automatically. When your question matches a skill's topic, the agent reads it and applies the guidance for that session. No manual activation is needed.

## Contributions and Improvements

Open for improvements and more skills. Contributions and feedback welcome via pull requests.

Inspired by and modeled on the [MariaDB Agent Skills](https://github.com/MariaDB/skills) format.
