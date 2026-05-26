---
name: percona-operator-for-mysql
description: "Running MySQL on Kubernetes with the Percona Operators. Use when deploying MySQL/Percona Server/Percona XtraDB Cluster on Kubernetes, choosing a Percona MySQL operator, writing the cluster Custom Resource (CR), configuring backups to S3/GCS/Azure, scaling, or upgrading a Kubernetes MySQL deployment. IMPORTANT — there are TWO distinct Percona MySQL operators: one based on Percona XtraDB Cluster (PXC, Galera synchronous, mature/GA) and one based on Percona Server (PS, Group Replication GA + async tech-preview). They have separate CRDs (pxc.percona.com vs ps.percona.com) and are NOT interchangeable. Always change the CR, never edit pods or StatefulSets directly; the operator reconciles and overwrites manual changes."
---

# Percona Operators for MySQL on Kubernetes

*Last updated: 2026-05-26*

A Kubernetes operator is a controller that manages an application's full lifecycle through Custom Resources. Percona ships **two distinct MySQL operators**, and the most common mistake is treating them as one:

- **Operator for MySQL based on Percona XtraDB Cluster (PXC)** — Galera synchronous multi-primary, HAProxy/ProxySQL proxy. Mature, GA, supports cross-site replication and PITR in production.
- **Operator for MySQL based on Percona Server (PS)** — built on stock Percona Server, **Group Replication (GA)** or asynchronous (**tech preview**), HAProxy/MySQL Router proxy, Orchestrator for failover.

Both automate deployment, failover, backups (via XtraBackup), scaling, upgrades, and TLS — all declared in the cluster CR.

> **Versions (latest in scope):** PXC operator **v1.19.1** (manages PXC 8.4 / 8.0 / 5.7); PS operator **v1.1.0** (manages Percona Server 8.4 / 8.0). The PS operator's **async replication is tech preview** as of v1.1.0 — not for production; Group Replication is GA. Set `crVersion` in the CR to match the installed operator. Confirm current versions at the docs.

## What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| "The Percona MySQL Operator" as one product; PXC CRDs applied to a PS install | Two operators, separate CRDs (`pxc.percona.com` vs `ps.percona.com`). Check which is installed before applying manifests |
| Deploying MySQL via a plain StatefulSet + init scripts | Use the operator CR — it handles cluster formation, failover, backups, TLS, and rolling upgrades correctly |
| `kubectl exec` into a pod to edit `my.cnf` | Configuration goes in the CR; the operator reconciles and overwrites manual pod changes |
| `kubectl scale sts ...` to add a node | Change `spec.pxc.size` (PXC) / `spec.mysql.size` (PS); the operator does safe SST / GR member changes |
| `size: 4` for a PXC cluster | Use odd sizes (3 or 5) — even counts risk split-brain; rejected unless `unsafeFlags.pxcSize: true` |
| Treating PS-operator async replication as production-ready | It is **tech preview** (v1.1.0). Group Replication is the GA topology |
| Choosing the PS operator for cross-site / multi-cluster DR | Multi-cluster replication is supported by the **PXC** operator |
| Backup storage without a credentials Secret | S3/GCS/Azure storage needs a Kubernetes Secret with credentials, referenced from the CR |
| Wrong Helm chart for the operator type | PXC: `percona/pxc-operator` + `percona/pxc-db`; PS: `percona/ps-operator` + `percona/ps-db` |
| Stale/missing `crVersion` after an operator upgrade | `crVersion` must match the operator version or reconciliation misbehaves |

## Choosing the Operator

| | PXC-based | PS-based |
|---|---|---|
| Replication | Galera **synchronous** multi-primary | **Group Replication** (GA) / async (tech preview) |
| Proxy | HAProxy, ProxySQL | HAProxy, MySQL Router |
| Maturity | GA since 2019, long track record | Newer; GR GA, async tech preview |
| Cross-site / multi-cluster | Supported | Not supported |
| Choose when | You want proven synchronous HA, cross-site DR, production PITR | You want stock Percona Server + Group Replication + MySQL Router |

## Minimal PXC Cluster CR

```yaml
apiVersion: pxc.percona.com/v1
kind: PerconaXtraDBCluster
metadata:
  name: cluster1                 # ≤22 chars
  finalizers: [percona.com/delete-pods-in-order]
spec:
  crVersion: "1.19.1"            # match the installed operator
  secretsName: cluster1-secrets
  updateStrategy: SmartUpdate
  pxc:
    size: 3                      # odd: 3 or 5
    image: percona/percona-xtradb-cluster:8.4.7-7.1
    resources: { requests: { memory: 1G, cpu: 600m } }
    volumeSpec:
      persistentVolumeClaim:
        storageClassName: standard
        resources: { requests: { storage: 6Gi } }
  haproxy:                        # enable HAProxy OR proxysql, not both
    enabled: true
    size: 3
  pmm:
    enabled: false                # true to attach PMM monitoring
  backup:
    enabled: true
    storages:
      s3-backup:
        type: s3
        s3:
          bucket: my-backup-bucket
          region: us-west-2
          credentialsSecret: cluster1-backup-s3   # Secret with cloud creds
    schedule:
      - name: daily
        schedule: "0 2 * * *"
        storageName: s3-backup
        retention: { count: 7, type: count, deleteFromStorage: true }
    pitr:
      enabled: false              # true for point-in-time recovery (PXC: GA)
      storageName: s3-backup
      timeBetweenUploads: 60
```
CR kinds: `PerconaXtraDBCluster` / `PerconaXtraDBClusterBackup` / `…Restore` (PXC); `PerconaServerMySQL` / `PerconaServerMySQLBackup` / `…Restore` (PS).

## Install

```bash
# Helm (recommended) — PXC
helm repo add percona https://percona.github.io/percona-helm-charts/ && helm repo update
helm install my-op percona/pxc-operator -n pxc --create-namespace
helm install my-db percona/pxc-db -n pxc
# PS operator uses percona/ps-operator + percona/ps-db

# Or kubectl + bundle.yaml (PXC)
kubectl apply --server-side -f https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/v1.19.1/deploy/bundle.yaml -n pxc
kubectl apply -f https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/v1.19.1/deploy/cr.yaml -n pxc

kubectl get pxc -n pxc        # status (use 'kubectl get ps' for the PS operator)
```
OpenShift: both operators install via OperatorHub / OLM (PXC operator is Red Hat Certified).

## Backups

Both operators back up with **Percona XtraBackup** to S3, Azure Blob, or (PS operator / on-prem PVC) GCS. Scheduled backups are defined in `spec.backup.schedule[]` (cron) with a retention policy; on-demand backups are a separate `…Backup` CR (`kubectl apply -f backup.yaml`). **PITR**: GA on the PXC operator (dedicated pod streams binlogs to cloud storage; incompatible with count retention — use bucket lifecycle); tech preview on the PS operator (via Percona Binlog Server). Incremental backups are tech preview on the PS operator (v1.1.0).

## Sources

- [Compare the operators](https://docs.percona.com/percona-operator-for-mysql/pxc/compare.html)
- PXC operator: [architecture](https://docs.percona.com/percona-operator-for-mysql/pxc/architecture.html) · [helm](https://docs.percona.com/percona-operator-for-mysql/pxc/helm.html) · [backups](https://docs.percona.com/percona-operator-for-mysql/pxc/backups.html)
- PS operator: [how it works](https://docs.percona.com/percona-operator-for-mysql/ps/how-it-works.html) · [backups](https://docs.percona.com/percona-operator-for-mysql/ps/backups.html)

*Replace versions/paths with your installed operator. For anything not covered here, see [docs.percona.com/percona-operator-for-mysql](https://docs.percona.com/percona-operator-for-mysql).*
