# MongoDB replica set (local)

1 **primary** + 2 **secondaries** = 3 pods using Bitnami `architecture: replicaset`.

Config: [values.yaml](./values.yaml) (replica set is the default). API URI: [grade-submission-api/values.yaml](../grade-submission-api/values.yaml).

See also: [prod/mongo.md](../../prod/mongo.md)

Check latest chart version:

```bash
helm repo update bitnami
helm search repo bitnami/mongodb --versions | head -5
```

Tested with chart **19.1.14** (MongoDB 8.3.x, default Bitnami image — no `bitnamilegacy` override).

---

## Architecture

```mermaid
flowchart TB
  API[API pods - mongoose]
  HL[mongodb-headless Service]
  P0["mongodb-0 PRIMARY"]
  P1["mongodb-1 SECONDARY"]
  P2["mongodb-2 SECONDARY"]
  PVC0["PVC datadir-mongodb-0"]
  PVC1["PVC datadir-mongodb-1"]
  PVC2["PVC datadir-mongodb-2"]

  API -->|"MONGODB_URI (stateless-v4)"| HL
  HL --> P0
  HL --> P1
  HL --> P2
  P0 --> PVC0
  P1 --> PVC1
  P2 --> PVC2
  P0 -.->|replication| P1
  P0 -.->|replication| P2
```

Each pod has **its own PVC**. Data is copied between nodes by **MongoDB replication**, not by sharing one volume.

Kubernetes workload: **StatefulSet** (not Deployment). Verify without installing:

```bash
helm template mongodb bitnami/mongodb \
  --version 19.1.14 \
  -f values.yaml \
  -n grade-submission | grep 'kind: StatefulSet\|kind: Deployment'
# → kind: StatefulSet
```

See [helm_commands.md](../../helm_commands.md#template-render-manifests).

---

## Bitnami: `architecture` vs `useStatefulSet`

Chart defaults (`helm show values bitnami/mongodb`) use `architecture: standalone` and `useStatefulSet: false`. That is **not** what this repo installs — [values.yaml](./values.yaml) sets `architecture: replicaset`.

| `architecture` | `useStatefulSet` | K8s workload | MongoDB topology |
|------------------|------------------|--------------|------------------|
| `standalone` | `false` (chart default) | **Deployment** | single node, no replica set |
| `standalone` | `true` | **StatefulSet** (1 pod) | single node, stable name + PVC |
| **`replicaset`** | *ignored* | **StatefulSet** (N pods) | replica set — our setup |

`useStatefulSet` applies **only when** `architecture=standalone`. With `architecture: replicaset`, the chart always renders a StatefulSet; you do not set `useStatefulSet`.

Compare modes locally:

```bash
# chart default — Deployment
helm template test bitnami/mongodb --version 19.1.14 \
  --set architecture=standalone --set useStatefulSet=false \
  | grep 'kind: StatefulSet\|kind: Deployment'

# standalone but StatefulSet — one pod, mongodb-0
helm template test bitnami/mongodb --version 19.1.14 \
  --set architecture=standalone --set useStatefulSet=true \
  | grep 'kind: StatefulSet\|kind: Deployment'

# our values — StatefulSet, 3 pods, replica set
helm template test bitnami/mongodb --version 19.1.14 -f values.yaml \
  | grep 'kind: StatefulSet\|kind: Deployment'
```

After install, confirm in the cluster:

```bash
kubectl get statefulset,deployment -n grade-submission -l app.kubernetes.io/name=mongodb
# mongodb   3/3   (StatefulSet only — no Deployment)
```

Hand-written example (course material): [09_hpa/mongodb/mongodb-statefulset.yaml](../../09_hpa/mongodb/mongodb-statefulset.yaml) — same StatefulSet + `volumeClaimTemplates` pattern, but **without** Bitnami’s replica set bootstrap.

---

## PVCs in a replica set

| Pod | PVC name (Bitnami) | Role |
|-----|-------------------|------|
| `mongodb-0` | `datadir-mongodb-0` | usually starts as primary |
| `mongodb-1` | `datadir-mongodb-1` | secondary |
| `mongodb-2` | `datadir-mongodb-2` | secondary |

**Important:**

- **One PVC per pod** — three replicas ⇒ three PVCs.
- **`helm uninstall` does not delete PVCs** — all three disks remain with data.
- **Deleting one PVC** — see [What happens if you delete one PVC?](#what-happens-if-you-delete-one-pvc) below.
- **Changing topology** — not in-place; always uninstall + delete PVCs + reinstall.

List PVCs:

```bash
kubectl get pvc -n grade-submission | grep mongodb
```

### What happens if you delete one PVC?

Example: you delete `datadir-mongodb-1` while the replica set is running.

1. Pod `mongodb-1` loses its disk and is recreated with an **empty** PVC.
2. MongoDB treats it as a **new or recovering member** — the primary **starts syncing** (copying data) to that secondary again.
3. Until sync completes, that member may show `STARTUP2`. The replica set can still serve traffic if enough members remain.
4. If you delete the **primary’s** PVC, MongoDB holds an **election** and a secondary is promoted.

---

## Install (from scratch)

### 1. Tear down existing MongoDB (if any)

```bash
helm uninstall mongodb -n grade-submission
kubectl delete pvc datadir-mongodb-0 datadir-mongodb-1 datadir-mongodb-2 -n grade-submission
# NotFound on PVCs that never existed is OK
```

Verify clean:

```bash
helm list -n grade-submission | grep mongodb
kubectl get pods,pvc,svc -n grade-submission | grep mongodb
```

### 2. Install replica set

```bash
cd 12_helm_mongo/mongodb
helm repo update bitnami
helm install mongodb bitnami/mongodb \
  --version 19.1.14 \
  -f values.yaml \
  -n grade-submission \
  --create-namespace
```

Pods start **one by one** (`OrderedReady`) — can take several minutes:

```bash
kubectl get pods -n grade-submission -w
```

Expected (no `mongodb-arbiter` — disabled in `values.yaml`):

```text
mongodb-0   1/1   Running
mongodb-1   1/1   Running
mongodb-2   1/1   Running
```

### 3. Verify replica set

```bash
kubectl exec -n grade-submission mongodb-0 -- \
  mongosh -u admin -p password123 --eval "rs.status().members.map(m => ({name: m.name, state: m.stateStr}))"
```

```bash
kubectl get pvc -n grade-submission
kubectl get svc -n grade-submission | grep mongodb
```

### 4. Upgrade API (`stateless-v4` + replica set `MONGODB_URI`)

URI is in [grade-submission-api/values.yaml](../grade-submission-api/values.yaml) (plain text in comment, base64 in `secrets.MONGODB_URI`).

```bash
cd 12_helm_mongo
helm upgrade grade-submission-api ./grade-submission-api \
  -n grade-submission \
  -f grade-submission-api/values.yaml \
  --reset-values
```

---

## API connection — `stateless-v4`

**`stateless-v4`** uses one env var:

```javascript
const MONGODB_URI = process.env.MONGODB_URI || 'mongodb://localhost:27017/grades';
mongoose.connect(MONGODB_URI, { ... });
```

Default replica set URI (in comments in `grade-submission-api/values.yaml`):

```text
mongodb://admin:password123@mongodb-0.mongodb-headless.grade-submission.svc.cluster.local:27017,mongodb-1.mongodb-headless.grade-submission.svc.cluster.local:27017,mongodb-2.mongodb-headless.grade-submission.svc.cluster.local:27017/?replicaSet=rs0
```

Optional read from secondaries (comment only — re-encode `MONGODB_URI` if enabled):

```text
.../?replicaSet=rs0&readPreference=secondaryPreferred
```

Re-encode:

```bash
echo -n 'mongodb://admin:password123@mongodb-0.mongodb-headless.grade-submission.svc.cluster.local:27017,mongodb-1.mongodb-headless.grade-submission.svc.cluster.local:27017,mongodb-2.mongodb-headless.grade-submission.svc.cluster.local:27017/?replicaSet=rs0' | base64
```

---

## Read / write routing

**Not explicitly configured.** Writes → primary. Reads → primary (driver default). Failover → yes, via `replicaSet=rs0` in URI.

| Operation | Default route |
|-----------|---------------|
| Writes (`save`) | Primary only |
| Reads (`find`) | Primary (`readPreference: primary`) |
| Failover | Driver finds new primary after election |

To read from secondaries, add to URI (see comment in `grade-submission-api/values.yaml`):

```text
.../?replicaSet=rs0&readPreference=secondaryPreferred
```

---

## Useful commands

```bash
kubectl get pods -n grade-submission -l app.kubernetes.io/name=mongodb
kubectl logs mongodb-0 -n grade-submission
kubectl exec -it mongodb-0 -n grade-submission -- mongosh -u admin -p password123
kubectl exec -n grade-submission mongodb-0 -- \
  mongosh -u admin -p password123 --eval "rs.conf()"

helm uninstall mongodb -n grade-submission
kubectl delete pvc datadir-mongodb-0 datadir-mongodb-1 datadir-mongodb-2 -n grade-submission
```

---

## Notes

- **Chart 15.x + `bitnamilegacy`** had replica set bootstrap issues locally — use **chart 19.x** with default image (current `values.yaml`).
- **11_helm** custom chart does not configure a real replica set — use Bitnami for this exercise.
