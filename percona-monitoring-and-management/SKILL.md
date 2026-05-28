---
name: percona-monitoring-and-management
description: "Open-source database observability for MySQL/Percona Server/PXC, PostgreSQL, and MongoDB with Percona Monitoring and Management (PMM). Use when setting up monitoring for MySQL or Percona Server, analyzing query performance continuously, choosing between PMM and a hand-built Prometheus/Grafana stack, configuring Query Analytics (QAN), or registering database instances with a monitoring server. IMPORTANT — PMM is client/server: PMM Server runs centrally (Docker/Helm/appliance) and a PMM Client (pmm-agent) runs on each database host. QAN needs the slow query log or Performance Schema enabled on the database; it is not automatic. Confirm whether the deployment is PMM 2.x or PMM 3.x — image tags, client packages, and doc URLs differ."
---

# Percona Monitoring and Management (PMM)

*Last updated: 2026-05-26*

PMM is Percona's open-source observability platform for databases. It is **client/server**: **PMM Server** (Docker container, Helm chart, or VM appliance) stores metrics in VictoriaMetrics and query data in ClickHouse, and serves Grafana dashboards plus Query Analytics (QAN); a **PMM Client** (`pmm-agent` + exporters) runs on every monitored host and pushes data to the server. It covers MySQL, Percona Server, PXC, PostgreSQL, MongoDB, and ProxySQL — a unified, supported alternative to assembling exporters, Prometheus, and Grafana yourself.

> **Version note:** **PMM 3.x** is the current major release (GA 2025); **PMM 2.x** is the prior line. The concepts below are stable across both, but **image tags, client packages, and doc URL paths differ** — `percona/pmm-server:3` vs `:2`, `pmm-client` (v3) vs `pmm2-client`, and `/percona-monitoring-and-management/3/…` vs `/2/…` in doc URLs. There is no in-place upgrade across major lines (1→2 or 2→3) — stand up the new server and re-register clients. Confirm the running version before copying commands; the examples below show the PMM 3 form.

## What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| Hand-rolling Prometheus + Grafana + exporters for MySQL | PMM bundles VictoriaMetrics, ClickHouse, Grafana, QAN, and Percona dashboards as one supported stack — and gives you query analytics you'd otherwise build by hand |
| "PMM uses Prometheus internally" | VictoriaMetrics replaced Prometheus (since PMM 2.12). The `/prometheus`-style API stays for compatibility |
| Expecting QAN to work out of the box | QAN needs a data source enabled on the DB: slow query log **or** Performance Schema. Neither is fully configured by default |
| Installing PMM Client on the PMM Server host to watch each DB | `pmm-agent` runs on each **database** host, not on the server. (Exception: AWS RDS is monitored remotely with no client on the instance) |
| `--query-source=perfschema` for Percona Server | For Percona Server / PXC use the **slow log** (richer data, sampling); use Performance Schema for stock MySQL / MariaDB |
| Exposing PMM Server on plain HTTP, default creds | Use HTTPS (PMM 3: container port **8443**; PMM 2: **443**), change the default `admin` password immediately, replace the self-signed cert for production |
| Slow-log QAN with a remote PMM Client | Slow-log QAN needs the client on the **same host** (it reads the file). For remote-only DBs use Performance Schema |
| Mixing PMM 2 and PMM 3 packages/images | Match them: server `:3` ↔ `pmm-client` v3, server `:2` ↔ `pmm2-client` |
| `pt-query-digest` makes PMM QAN redundant | Complementary: `pt-query-digest` is one-shot offline log analysis; QAN is continuous, stored, interactive |

## Architecture (what to reason about)

- **PMM Server** — `pmm-managed` (API/orchestration), **VictoriaMetrics** (metrics), **ClickHouse** (QAN data), **Grafana** (dashboards), Alertmanager, an internal PostgreSQL, behind Nginx (TLS).
- **PMM Client** per DB host — `pmm-agent` (daemon), `pmm-admin` (CLI), `vmagent` (pushes metrics), and exporters (`mysqld_exporter`, `node_exporter`, etc.).
- **Data flow:** exporters → `vmagent` push → VictoriaMetrics; QAN agent collects query buckets each minute → ClickHouse.
- **Ports:** server UI/API on **8443/8080** in PMM 3 (was **443/80** in PMM 2), agent on 7777, exporter range 42000–51999.

## Install & Connect

```bash
# 1. PMM Server (Docker). Persist /srv on a named volume.
docker volume create pmm-data
docker run -d --restart always -p 8443:8443 -v pmm-data:/srv \
  --name pmm-server percona/pmm-server:3
docker exec -t pmm-server change-admin-password <new_password>

# 2. PMM Client on each DB host (Percona repo)
#    Debian/Ubuntu:
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo apt update && sudo apt install -y pmm-client    # pmm2-client for PMM 2.x

# 3. Register the node with the server
pmm-admin config --server-insecure-tls \
  --server-url=https://admin:<password>@<PMM_SERVER_IP>:8443   # PMM 3; use :443 for PMM 2
```

MySQL monitoring user:
```sql
CREATE USER 'pmm'@'127.0.0.1' IDENTIFIED BY '<pass>' WITH MAX_USER_CONNECTIONS 10;
GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD, BACKUP_ADMIN ON *.* TO 'pmm'@'127.0.0.1';
```
Add the instance (pick the QAN source):
```bash
# Percona Server / PXC — slow log (recommended)
pmm-admin add mysql --query-source=slowlog --size-slow-logs=1GiB \
  --username=pmm --password=<pass> my-mysql 127.0.0.1:3306
# Stock MySQL — Performance Schema
pmm-admin add mysql --query-source=perfschema --username=pmm --password=<pass> my-mysql
pmm-admin inventory list services --service-type=mysql   # verify
```

## Query Analytics (QAN)

QAN aggregates queries by fingerprint into per-minute buckets, with latency breakdowns, example queries, and EXPLAIN — stored historically in ClickHouse. Two MySQL sources:

| | Slow query log | Performance Schema |
|---|---|---|
| Detail | Higher (esp. with Percona's `log_slow_verbosity='full'`) | Lower |
| Client location | Same host as DB (reads the file) | Can be remote |
| Best for | Percona Server, PXC | Stock MySQL 8.0+, MariaDB |

Slow-log config for QAN on Percona Server:
```ini
slow_query_log = ON
log_output = FILE
long_query_time = 0
log_slow_admin_statements = ON
log_slow_verbosity = full
log_slow_rate_limit = 100        # sample 1% on a busy primary
slow_query_log_use_global_control = all
```
QAN is the continuous counterpart to `pt-query-digest` (see `percona-toolkit`): use QAN for ongoing monitoring, `pt-query-digest` for ad-hoc forensic dives on a captured log.

## pmm-admin Reference

```bash
pmm-admin add postgresql --username=pmm --password=*** --environment=prod --cluster=pg1 my-pg 127.0.0.1:5432
pmm-admin add mongodb --username=admin --password=*** --replication-set=rs0 my-mongo 127.0.0.1:27017
pmm-admin list                       # services on this node
pmm-admin remove mysql my-mysql      # stop monitoring a service
pmm-admin status                     # agent connectivity / server version
```
The `--environment`, `--cluster`, `--replication-set`, and `--custom-labels` flags (on every `add`) drive filtering and grouping in the UI — set them consistently.

## Alerting, Advisors, Auth

- **Percona Alerting** (PMM 2.31+; replaces the old "Integrated Alerting") — build alert rules from Percona-supplied templates (MetricsQL), or wire an external Alertmanager. Don't reach for "Integrated Alerting"; it's gone.
- **Advisors** — automated Security/Configuration/Performance/Query health checks that run on a schedule (default 24h). A free, Percona-specific source of "what's wrong with this server" that agents never think to mention.
- **Auth:** PMM 3 uses Grafana **service-account tokens**; PMM 2 uses **API keys** (auto-migrated on upgrade). Always change the default `admin` password and front the server with TLS.
- **Upgrades:** within a major line, upgrade via the Home-dashboard update panel (server **before** clients). Across majors (2→3) there's no in-place upgrade — deploy a fresh PMM 3 server and re-register clients with the `pmm-client` (v3) package.

## Sources

- [PMM documentation home](https://docs.percona.com/percona-monitoring-and-management/) — pick your version (3 or 2)
- [Architecture](https://docs.percona.com/percona-monitoring-and-management/3/reference/index.html)
- [Set up PMM Server (Docker)](https://docs.percona.com/percona-monitoring-and-management/3/install-pmm/install-pmm-server/deployment-options/docker/index.html)
- [Add MySQL services](https://docs.percona.com/percona-monitoring-and-management/3/install-pmm/install-pmm-client/index.html)
- [Query Analytics](https://docs.percona.com/percona-monitoring-and-management/3/use/qan/index.html)

*Doc paths use `/3/` for PMM 3 and `/2/` for PMM 2 — adjust to your version. For anything not covered here, see [docs.percona.com/percona-monitoring-and-management](https://docs.percona.com/percona-monitoring-and-management/).*
