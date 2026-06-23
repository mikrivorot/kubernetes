# MongoDB in production

Short overview of how production differs from local setups in [11_helm](../11_helm/) and [12_helm_mongo](../12_helm_mongo/).

Local replica set guide (commands + PVCs): [12_helm_mongo/mongodb/replicaset.md](../12_helm_mongo/mongodb/replicaset.md)

## What you do locally

- **11_helm** — custom mongo chart, single node (no replica set)
- **12_helm_mongo** — Bitnami **replica set** (3 data nodes), chart 19.x, [values.yaml](../12_helm_mongo/mongodb/values.yaml)

Fine for learning on Docker Desktop — not a copy-paste production plan.

---

## Production options

### 1. Managed MongoDB (most common)

Use a cloud service: **MongoDB Atlas**, **Azure Cosmos DB (Mongo API)**, or similar.

- **Replica set / HA built in** — provider runs primary + secondaries across zones
- Backups, failover, patching handled for you
- App gets a connection string (often SRV URI with `replicaSet` and TLS)
- Deploy config via Terraform, Helm values, or GitOps — not manual `helm install` from a laptop

**Best default for real production.** You do not manage PVCs or pod failover yourself.

### 2. Helm / Operator on Kubernetes (self-managed cluster)

Run MongoDB on K8s via **MongoDB Community Operator**, **Percona Operator**, or a pinned Helm chart — deployed by **Terraform** or **GitOps**.

- You define replica count, storage class, resources in `values-prod.yaml`
- **Each pod gets its own PVC** (see below)
- You own backups, monitoring, upgrades, and disaster recovery

**Note:** Bitnami public images moved to `bitnamilegacy` — weak default for new prod; prefer managed DB or an operator with supported images.

### 3. Your own manifests / chart (like 11_helm/mongodb)

Store StatefulSet, Service, Secret, PVC templates in **your repo**.

- Full control, same git workflow as the app
- **You** must implement replica set bootstrap, TLS, and ops runbooks

Good for learning; heavy for most app teams unless you have a platform/DBA function.

---

## Replica set in production (1 primary + 2 secondaries)

MongoDB HA is a **replica set**: one **primary** (writes) and **secondaries** (replicate + can be promoted on failure).

| | Local (Bitnami replica set on Docker Desktop) | Production |
|---|-----------------------------------------------|------------|
| Nodes | 3 data pods on **one** machine | 3+ pods across **availability zones** |
| Primary election | **Automatic** (MongoDB replica set) | **Automatic** (same mechanism) |
| Survives node / zone loss | **No** — one Docker Desktop node | **Yes** — if members span zones |
| Storage | `hostpath`, single machine | **Regional disks** (e.g. GCE PD, EBS) per pod |
| Connection | Replica-set URI (`MONGODB_URI`, `replicaSet=rs0`) | Replica-set URI + **TLS**, often SRV |
| Who runs it | You via Helm | **Atlas / operator / platform team** |

Local and prod both use **automatic primary election** when configured as a replica set. The gap is **where** nodes run (one laptop vs multi-zone), **storage**, **TLS**, and **ops** — not the election algorithm itself.

**Typical prod connection** (app or Secret):

```text
mongodb+srv://user:pass@cluster.example.mongodb.net/mydb?retryWrites=true&w=majority
```

Or non-SRV with explicit hosts and `replicaSet=...`.

The course API **`stateless-v4`** uses `MONGODB_URI` (full string with `replicaSet=...`). 

**`stateless-v3`** uses separate host/user/password — single host only, not enough for prod failover.

---

## PVCs and the cluster (replica set)

In Kubernetes, a MongoDB replica set on StatefulSet looks like this:

```
StatefulSet (mongodb)
  ├── Pod mongodb-0  →  PVC datadir-mongodb-0  →  PV (disk A)
  ├── Pod mongodb-1  →  PVC datadir-mongodb-1  →  PV (disk B)
  └── Pod mongodb-2  →  PVC datadir-mongodb-2  →  PV (disk C)
```

### Rules

1. **One PVC per pod** — replicas do **not** share one volume. Replication is inside MongoDB, not at the filesystem level.
2. **Pod name is stable** — `mongodb-0` always maps to `datadir-mongodb-0` after rescheduling (same data on same disk).
3. **Headless Service** — DNS like `mongodb-1.mongodb-headless.namespace.svc.cluster.local` lets MongoDB and clients reach each member.
4. **`helm uninstall` removes pods, not PVCs** — data survives until you delete PVCs or rely on backup/retention policy.
5. **Production storage** — use a **StorageClass** with replicated/block storage in the cloud; avoid `hostpath` (single-node, not portable).

### Local vs prod PVC behaviour

| Action | Local (Docker Desktop) | Production |
|--------|------------------------|------------|
| Delete pod | Pod recreated, **same PVC** reattached | Same — data kept |
| Delete PVC | Data gone on that node | Data gone unless snapshots/backups exist |
| `helm uninstall` | 3 PVCs may remain | Same — disks often retained by policy |
| Node loss | Rare on one-node desktop | New pod schedules elsewhere **only if** PV is multi-zone or restored from backup |

### Prod commands (conceptual — via GitOps/Terraform, not ad hoc)

```bash
# Inspect (read-only ops)
kubectl get statefulset,pods,pvc -n database
kubectl exec -n database mongodb-0 -- mongosh --eval "rs.status()"

# Never routine in prod: manual uninstall without backup plan
helm uninstall mongodb -n database
kubectl get pvc -n database   # disks often kept on purpose
```

Backups in prod: **Atlas continuous backup**, **Velero**, **volume snapshots**, or **mongodump** cron — not “hope the PVC is still there.”

---

## Replication is standard — StatefulSet is optional

Two different layers:

```mermaid
flowchart TB
  subgraph mongo ["MongoDB (same local and prod)"]
    RS[Replica set]
    P[Primary]
    S1[Secondary]
    S2[Secondary]
    RS --> P
    RS --> S1
    RS --> S2
    P -.->|ongoing replication| S1
    P -.->|ongoing replication| S2
  end

  subgraph k8s ["Kubernetes (only if DB runs on your cluster)"]
    SS[StatefulSet]
    SS --> P
    SS --> S1
    SS --> S2
  end

  subgraph managed ["Managed e.g. Atlas (common prod)"]
    AT[Provider runs replica set]
    AT --> RS
  end
```

### MongoDB replication (standard for HA)

**Yes — replication is the standard.** Production HA almost always means a **replica set**: one primary, secondaries copy the oplog, automatic election if the primary dies. Your local Bitnami setup uses the **same MongoDB feature** as Atlas or a bare-metal install.

Replication is **not** a Kubernetes feature — it happens **inside MongoDB** between members.

### StatefulSet (only when MongoDB runs on Kubernetes)

**Sometimes in prod — not always.**

| How prod runs MongoDB | Uses StatefulSet? | Who manages replication? |
|-----------------------|-------------------|---------------------------|
| **MongoDB Atlas / managed DB** | **No** (not in your cluster) | Provider — replica set on their infra |
| **Helm / operator on K8s** (Bitnami, Percona, Community Operator) | **Yes** | MongoDB replica set; K8s schedules pods + PVCs |
| **VMs / bare metal** | **No** | MongoDB replica set on machines |

StatefulSet gives **stable pod names** (`mongodb-0`, `mongodb-1`, …) and **one PVC per pod** — a good fit for stateful databases on K8s. It does **not** replace replication; it **hosts** replica set members.

### How they work together (self-hosted on K8s)

1. **StatefulSet** creates 3 pods, each with its own PVC.
2. **Bitnami chart / operator** bootstraps a **MongoDB replica set** across those pods.
3. **MongoDB** replicates data primary → secondaries (standard oplog sync).
4. **Your app** connects with a replica-set URI; the driver handles failover after election.

On Docker Desktop all 3 pods sit on one node — replication works, but you lose everything if that node dies. In prod, members are spread across zones **and** replication still works the same way.

### Bitnami chart: `useStatefulSet: false` in defaults

`helm show values bitnami/mongodb` lists `architecture: standalone` and `useStatefulSet: false`. That means the **chart default** is a **Deployment** (one pod, no replica set) — not what we use locally.

Our [values.yaml](../12_helm_mongo/mongodb/values.yaml) sets `architecture: replicaset`. Then **`useStatefulSet` is ignored** and the chart always creates a **StatefulSet** (3 pods for HA).

| Mode | K8s workload |
|------|--------------|
| Bitnami default (`standalone`, `useStatefulSet: false`) | Deployment |
| `standalone` + `useStatefulSet: true` | StatefulSet (1 pod) |
| **`replicaset`** (local + many self-hosted prod setups) | StatefulSet (N pods) |

Check without installing: `helm template` — see [helm_commands.md](../helm_commands.md#template-render-manifests) and [replicaset.md](../12_helm_mongo/mongodb/replicaset.md#bitnami-architecture-vs-usestatefulset).

---

## Comparison

| | Local (`12_helm_mongo`) | Production |
|---|-------------------------|------------|
| **Replication** | **Yes** — MongoDB replica set (standard) | **Yes** — same mechanism |
| **Primary election** | Automatic | Automatic |
| **Install** | `helm install` by hand | Terraform / GitOps / provider console |
| **Runs where** | 3 pods on one Docker Desktop node | Managed service **or** K8s StatefulSet **or** VMs |
| **StatefulSet** | Yes (Bitnami chart) | Only if DB is on **your** Kubernetes cluster |
| **Survives zone loss** | No | Yes (if members span AZs) |
| **Storage** | `hostpath`, single machine | Cloud disk per pod / provider storage |
| **Connection** | `MONGODB_URI` + `replicaSet=rs0` | Same + TLS; often Atlas SRV |
| **Backups** | Manual (delete PVC = data gone) | Provider backups / snapshots |

---

## Practical recommendation

1. **Prod:** managed MongoDB replica set (Atlas etc.) — replication included, no StatefulSet in your cluster  
2. **Staging:** same as prod, or operator/Helm replica set on K8s with real StorageClass  
3. **Local:** Bitnami replica set in [12_helm_mongo](../12_helm_mongo/) — same replication model, limited by single-node Docker Desktop  

“Own files” in prod means **your values, secrets, and pipeline in git** — not necessarily avoiding Helm or StatefulSet. Managed DB means you skip both and still get standard MongoDB replication.
