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

*Replace `8.4` with your major version. For anything not covered here, see [docs.percona.com/percona-xtradb-cluster](https://docs.percona.com/percona-xtradb-cluster).*
