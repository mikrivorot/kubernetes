# Grade submission + Bitnami MongoDB

Same app charts as [11_helm](../11_helm/), but MongoDB comes from the **Bitnami** chart instead of the custom `11_helm/mongodb` chart.

## Key difference from 11_helm

| | 11_helm | 12_helm_mongo |
|---|---------|---------------|
| MongoDB | Custom chart, single node | Bitnami **replica set** (3 pods) |
| API connection | `MONGODB_HOST` + user/pass (`stateless-v3`) | `MONGODB_URI` (`stateless-v4`) |

Install everything in **`grade-submission`** namespace. Release name **`mongodb`**.

## Setup

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/mongodb --versions
```

## Install MongoDB (Bitnami replica set)

From `mongodb/`. See [values.yaml](./mongodb/values.yaml) and [mongodb/replicaset.md](./mongodb/replicaset.md) (`architecture: replicaset` → StatefulSet; chart default `useStatefulSet: false` applies only to standalone).

Preview manifests before install: [helm template](../../helm_commands.md#template-render-manifests).

```bash
cd mongodb
helm install mongodb bitnami/mongodb \
  --version 19.1.14 \
  -f values.yaml \
  -n grade-submission \
  --create-namespace
```

## Install app charts

```bash
helm install grade-submission-api ./grade-submission-api -n grade-submission
helm install grade-submission-portal ./grade-submission-portal -n grade-submission
```

Or upgrade:

```bash
helm upgrade grade-submission-api ./grade-submission-api -n grade-submission -f grade-submission-api/values.yaml --reset-values
```

## Chart vs container image

| In the command | Meaning |
|----------------|---------|
| `bitnami/mongodb` | Helm chart |
| `--version 19.1.14` | Chart version (check latest with `helm search`) |
| `-f values.yaml` | Auth + replica set overrides |

Default chart image is used (no `bitnamilegacy` in `values.yaml`). If ImagePullBackOff on older chart versions, upgrade to chart 19.x.

## URL structure

The API (`stateless-v4`) uses one env var: **`MONGODB_URI`**. Default in [grade-submission-api/values.yaml](./grade-submission-api/values.yaml):

```text
mongodb://admin:password123@mongodb-0.mongodb-headless.grade-submission.svc.cluster.local:27017,mongodb-1.mongodb-headless.grade-submission.svc.cluster.local:27017,mongodb-2.mongodb-headless.grade-submission.svc.cluster.local:27017/?replicaSet=rs0
```

### MongoDB URI parts

```text
mongodb://admin:password123@<host1>:27017,<host2>:27017,<host3>:27017/?replicaSet=rs0
│        │     │             │ hosts (comma-separated, one per replica)      │ query
│        │     └─ password   └─ port 27017 on each member                    └─ replica set name (Bitnami default: rs0)
│        └─ username (matches mongodb/values.yaml auth)
└─ scheme
```

| Part | Value here | Why |
|------|------------|-----|
| Credentials | `admin` / `password123` | Same as [mongodb/values.yaml](./mongodb/values.yaml) |
| Three hosts | `mongodb-0`, `mongodb-1`, `mongodb-2` | One StatefulSet pod per replica set member — driver needs all for discovery and failover |
| Port | `27017` | MongoDB default |
| `replicaSet=rs0` | Required | Tells the driver this is a replica set, not three standalone servers |

**Standalone fallback** (single Service, no replica set): `mongodb://admin:password123@mongodb:27017/` — see comment in `values.yaml`.

### Kubernetes DNS (one host)

Each host is a **pod DNS name** via the **headless Service** Bitnami creates:

```text
mongodb-0.mongodb-headless.grade-submission.svc.cluster.local
   │           │               │                │
   │           │               │                └─ cluster internal domain
   │           │               └─ namespace (API + MongoDB installed here)
   │           └─ headless Service (DNS → pod IP, not one shared ClusterIP)
   └─ pod name (StatefulSet ordinal; release name mongodb → mongodb-0, -1, -2)
```

General form: `<pod>.<service>.<namespace>.svc.cluster.local`

Inside the **same namespace**, the short form also works: `mongodb-0.mongodb-headless`.

### Why headless, not `mongodb`?

| Service | Type | Use |
|---------|------|-----|
| `mongodb-headless` | Headless | Replica set URI — reach **each** pod by stable DNS name |
| `mongodb` | ClusterIP | Standalone — one hostname, one entry point |

```text
grade-submission (namespace)
├── grade-submission-api pods  →  MONGODB_URI
└── mongodb StatefulSet
    ├── mongodb-0  ─┐
    ├── mongodb-1  ─┼─  mongodb-headless Service
    └── mongodb-2  ─┘
```

More: [mongodb/replicaset.md](./mongodb/replicaset.md) (install, PVCs, read/write routing).
