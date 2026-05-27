---
name: percona-xtradb-cluster
description: "Synchronous multi-master MySQL high availability with Percona XtraDB Cluster (PXC), built on Percona Server + Galera. Use when designing or operating a PXC cluster, choosing between PXC / async replication / Group Replication, bootstrapping or joining nodes, configuring SST/IST, running schema changes on a cluster, or debugging certification conflicts and quorum/split-brain. IMPORTANT — every table MUST have a primary key, only InnoDB is replicated, and transactions can fail at COMMIT (error 1213) due to certification conflicts, so applications must retry. PXC writes are bounded by the slowest node; it is not a write-scaling solution."
---

# Percona XtraDB Cluster (PXC)

*Last updated: 2026-05-26*

Percona XtraDB Cluster is Percona Server bundled with the Galera replication library, giving **synchronous, multi-master** clustering. A transaction's write-set is broadcast to all nodes and certified before commit: if it conflicts with a concurrent transaction on any node, the committing node rolls back. Every node is writable and stays consistent — a write confirmed anywhere is durable everywhere, or it never commits. The cost: commit latency grows with inter-node round-trips, and same-row writes from multiple nodes cause certification conflicts. PXC is for **high availability and read scaling with zero divergence**, not write scaling.

> **Versions:** PXC tracks Percona Server — **8.0** and **8.4 LTS** both use Galera 4. (No 9.x PXC at the time of writing — confirm at the docs.) Verify with `SELECT VERSION();` and `SHOW STATUS LIKE 'wsrep_provider_version';`.

## What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| Tables created without a PRIMARY KEY | Every table **must** have a PK — Galera needs it for write-set ordering. With `pxc_strict_mode=ENFORCING` (default), DML on PK-less tables is rejected |
| Assuming MyISAM data replicates | Only **InnoDB** is replicated. MyISAM writes are node-local by default (`wsrep_replicate_myisam=OFF`) → silent divergence |
| Treating `COMMIT` as guaranteed once DML succeeds | `COMMIT` can return `ERROR 1213` (certification conflict, not a local deadlock). The app must catch it and **retry the whole transaction** |
| Big `ALTER TABLE` in business hours with default DDL method | Default `wsrep_OSU_method=TOI` blocks **the whole cluster** for the DDL. Use RSU (rolling, per-node) or `pt-online-schema-change` for large changes |
| Disabling `pxc_strict_mode` to "make errors go away" | It is a correctness guard (PK/engine/locking checks). Fix the operation instead of disabling the net |
| Expecting writes to all nodes to scale throughput | Throughput is capped by the slowest node; concurrent same-row writes across nodes just raise the 1213 conflict rate |
| Leaving `innodb_autoinc_lock_mode=1` | Galera requires `=2` (interleaved). Strict mode aborts startup otherwise |
| `GET_LOCK()` / `LOCK TABLES` for cluster-wide coordination | These are node-local and unsupported cluster-wide; rejected in ENFORCING mode |
| Running a 2-node cluster for HA | A 2-node cluster loses quorum if one node fails. Use 3 nodes (or 2 + a `garbd` arbitrator) |
| Putting `wsrep_cluster_address=gcomm://` permanently in config to bootstrap | That makes every restart bootstrap a **new** cluster. Bootstrap only via `systemctl start mysql@bootstrap.service` |
| `wsrep_slave_threads` in new configs | Deprecated (since 8.0.26-16). Use `wsrep_applier_threads` |
| `wsrep_flow_control_paused > 0` read as a network problem | It means a node's apply queue is full and it's throttling all writers — check that node's I/O and applier threads |
| HAProxy `option mysql-check` / raw TCP health check | Use `option httpchk` with `clustercheck` (port 9200) so `JOINING`/`DONOR` nodes are excluded |
| `pc.bootstrap=true` on a survivor during a network partition | Only when the others are confirmed down — otherwise you get two diverging clusters needing full SST |
| Different SSL certs per node | All nodes (and garbd) must share identical CA/cert/key files or they won't join |

## Core Concepts

- **Write-set certification** — each commit is certified against concurrent transactions by primary key. Conflicts → rollback with error 1213 (optimistic concurrency, resolved at commit).
- **SST (State Snapshot Transfer)** — full data copy to a joining node. Default `wsrep_sst_method=xtrabackup-v2` (uses XtraBackup, donor stays online); `clone` available in 8.4.4+. Port 4444.
- **IST (Incremental State Transfer)** — sends only missing write-sets from the donor's **gcache** ring buffer (port 4568). Falls back to SST if the gap exceeds the gcache. Size it: `wsrep_provider_options="gcache.size=1G"`.
- **Quorum / split-brain** — a partition needs a strict majority to keep serving writes. Below quorum, nodes refuse writes (error 1047). `pc.weight` tunes weighted voting; recover a stuck partition with `SET GLOBAL wsrep_provider_options='pc.bootstrap=true'` on **one** side only.

## Bootstrapping & Joining

Config on every node (`mysqld.cnf`):
```ini
wsrep_provider=/usr/lib/galera4/libgalera_smm.so
wsrep_cluster_name=pxc-prod
wsrep_cluster_address=gcomm://10.0.0.1,10.0.0.2,10.0.0.3
wsrep_node_address=10.0.0.1          # unique per node
wsrep_node_name=pxc1                 # unique per node
wsrep_sst_method=xtrabackup-v2
pxc_strict_mode=ENFORCING
innodb_autoinc_lock_mode=2
default_storage_engine=InnoDB
wsrep_applier_threads=8
```
```bash
# Node 1 — bootstrap a NEW cluster (sets gcomm:// internally; stop with the same unit)
systemctl start mysql@bootstrap.service
#   verify: SHOW STATUS LIKE 'wsrep_cluster_size';  -> 1, wsrep_cluster_status=Primary

# Nodes 2 and 3 — normal start; SST happens automatically. Wait for Synced before adding the next.
systemctl start mysql
```
Open ports **3306** (SQL), **4567** (group comm), **4444** (SST), **4568** (IST). Total cluster restart: bootstrap the node with the highest `seqno` in `grastate.dat` (`safe_to_bootstrap: 1`); if all are `-1`, run `mysqld --wsrep-recover` to find the most advanced node.

## Schema Changes

- **TOI** (default) — DDL replicated in order; blocks the whole cluster for its duration. Fine for fast/instant DDL, dangerous for big rebuilds.
- **RSU** (`SET wsrep_OSU_method='RSU'`) — node desyncs, applies DDL locally, rejoins; cluster stays up but schemas differ transiently (the app must tolerate it). Needs gcache headroom.
- **pt-online-schema-change** — works on PXC with `wsrep_OSU_method=TOI`; preferred for large tables with active writes (see `percona-toolkit`).

## Key Variables

| Variable | Default | Purpose |
|---|---|---|
| `wsrep_cluster_address` | — | `gcomm://` peer list; empty = bootstrap |
| `wsrep_sst_method` | `xtrabackup-v2` | SST method (`clone` in 8.4.4+) |
| `pxc_strict_mode` | `ENFORCING` | PK/engine/locking validation (`DISABLED`/`PERMISSIVE`/`ENFORCING`/`MASTER`) |
| `wsrep_OSU_method` | `TOI` | DDL replication mode (`TOI`/`RSU`/`NBO`) |
| `wsrep_provider_options` | — | e.g. `gcache.size=1G;pc.weight=2` |
| `innodb_autoinc_lock_mode` | must be `2` | Required by Galera |
| `wsrep_applier_threads` | `1` | Parallel apply (tune to `wsrep_cert_deps_distance`) |
| `wsrep_sync_wait` | `0` | Causal-read consistency level |

## Monitoring Cluster Health

```sql
SHOW GLOBAL STATUS LIKE 'wsrep%';
```
| Variable | Healthy | Alarm |
|---|---|---|
| `wsrep_cluster_status` | `Primary` | anything else = not in the primary component |
| `wsrep_local_state_comment` | `Synced` | `Joining`/`Donor/Desynced` = not ready |
| `wsrep_ready` | `ON` | `OFF` → error 1047 on all DML |
| `wsrep_flow_control_paused` | ~`0` | `> 0.1` = a node is throttling the whole cluster |
| `wsrep_local_recv_queue_avg` | `0` | sustained `> 0` = this node can't keep up |
| `wsrep_local_cert_failures` / `wsrep_local_bf_aborts` | flat | rising = hot-row certification conflicts |

## Flow Control

When a slow node's receive queue hits the flow-control limit (`gcs.fc_limit`, config default 100 — the runtime value scales with node count unless `gcs.fc_master_slave=YES`) it sends `FC_PAUSE` and **every writer cluster-wide stalls** until it catches up. Diagnose:
```sql
SHOW STATUS LIKE 'wsrep_flow_control_paused';  -- fraction of time paused; >0.1 is bad
SHOW STATUS LIKE 'wsrep_flow_control_sent';    -- this node is the bottleneck if rising
```
Fix the slow node (disk I/O, raise `wsrep_applier_threads` toward `wsrep_cert_deps_distance`) — it is rarely a network issue.

## Load Balancing

Health-check on cluster state, not raw TCP, so traffic never goes to a `JOINING`/`DONOR` node.
- **HAProxy** — use `option httpchk` against the `clustercheck` script (port 9200), not `option mysql-check`.
- **ProxySQL** — add nodes to a hostgroup and set a monitor user; drain a node gracefully for maintenance with `SET GLOBAL pxc_maint_mode=MAINTENANCE;` (ProxySQL stops routing after ~10s), then `DISABLED` when done.

## garbd (Galera Arbitrator)

A voting-only daemon (no data) to give an even node count a quorum tiebreaker — cheaper than a third full node.
```ini
# /etc/default/garb
GALERA_NODES="10.0.0.1:4567,10.0.0.2:4567"
GALERA_GROUP="pxc-prod"
# SSL is ON by default in PXC 8.x — garbd MUST set a cipher or it crashes (gnu::NotSet):
GALERA_OPTIONS="socket.ssl_cipher=AES128-SHA256;socket.ssl_key=...;socket.ssl_cert=...;socket.ssl_ca=..."
```

## Encryption & Large Transactions

- **Traffic encryption** is ON by default (`pxc_encrypt_cluster_traffic=ON`, covers SST/IST/replication). All nodes must use **identical** cert files; changing it needs a full cluster stop.
- **Streaming replication** for huge transactions — by default the whole write-set is buffered in memory (`wsrep_max_ws_size` 2 GB). Fragment a big batch instead:
  ```sql
  SET SESSION wsrep_trx_fragment_unit='rows';
  SET SESSION wsrep_trx_fragment_size=10000;
  ```

## Troubleshooting & Upgrades

- **Error 1047 / node won't accept writes** — check `wsrep_cluster_status`/`wsrep_ready`. A minority partition refuses writes by design.
- **All nodes crashed (`seqno: -1`)** — run `mysqld --wsrep-recover` on each, bootstrap the highest seqno (`safe_to_bootstrap: 1`). Only use `SET GLOBAL wsrep_provider_options='pc.bootstrap=true'` when the other nodes are *confirmed down* — doing it during a mere network partition creates a split-brain.
- **Rolling 8.0 → 8.4 upgrade** — one node at a time (remove 8.0 packages, keep datadir, install 8.4, start → auto-upgrade + SST rejoin); keep writes on 8.0 nodes during the mixed window and avoid 8.4-only DDL until done. PXC 8.4 switches keyring plugin → component and defaults to `caching_sha2_password` (needs ProxySQL ≥ 2.6.2).

## PXC vs Async vs Group Replication

- **PXC** — strong consistency, no data loss on failover, automatic node provisioning; best within one low-latency datacenter; needs apps that retry on 1213. Min 3 nodes.
- **Async replication** — best for write-heavy single-primary, high-latency/cross-region, or bulk loads; replicas may lag and failover can lose data.
- **Group Replication** — MySQL-native alternative; PXC's strict mode **blocks running it alongside PXC**.

## Sources

- [PXC limitations](https://docs.percona.com/percona-xtradb-cluster/8.4/limitation.html)
- [Strict mode](https://docs.percona.com/percona-xtradb-cluster/8.4/strict-mode.html)
- [Bootstrap the cluster](https://docs.percona.com/percona-xtradb-cluster/8.4/bootstrap.html) · [Add nodes](https://docs.percona.com/percona-xtradb-cluster/8.4/add-node.html)
- [State Snapshot Transfer](https://docs.percona.com/percona-xtradb-cluster/8.4/state-snapshot-transfer.html)
- [Online schema upgrade (TOI/RSU/NBO)](https://docs.percona.com/percona-xtradb-cluster/8.4/online-schema-upgrade.html)
- [Crash recovery](https://docs.percona.com/percona-xtradb-cluster/8.4/crash-recovery.html)
- [Monitoring the cluster](https://docs.percona.com/percona-xtradb-cluster/8.4/monitoring.html)
- [Load balancing with ProxySQL](https://docs.percona.com/percona-xtradb-cluster/8.4/load-balance-proxysql.html) · [HAProxy](https://docs.percona.com/percona-xtradb-cluster/8.4/haproxy.html)
- [Galera Arbitrator (garbd)](https://docs.percona.com/percona-xtradb-cluster/8.4/garbd-howto.html) · [Encrypt traffic](https://docs.percona.com/percona-xtradb-cluster/8.4/encrypt-traffic.html)

*Replace `8.4` with your major version. For anything not covered here, see [docs.percona.com/percona-xtradb-cluster](https://docs.percona.com/percona-xtradb-cluster/).*
