# MongoDB in production

Short overview of how production differs from local setups in [11_helm](../11_helm/) and [12_helm_mongo](../12_helm_mongo/).

## What you do locally

- **11_helm** — custom mongo chart, Service name `mongodb`
- **12_helm_mongo** — Bitnami chart, release name `mongodb`, auth + `bitnamilegacy` image

Both are fine for learning and Docker Desktop. Do not treat them as a copy-paste production plan.

---

## Production options

### 1. Managed MongoDB (most common)

Use a cloud service: **MongoDB Atlas**, **Azure Cosmos DB (Mongo API)**, or similar.

- Backups, HA, patching handled by the provider
- App config stays the same shape: `MONGODB_HOST`, `MONGODB_USER`, `MONGODB_PASSWORD`, `MONGODB_PORT` (or a connection URI in Secret)
- Deploy config via Terraform, Helm values, or GitOps — not manual `helm install` from a laptop

**Best default for real production.**

### 2. Helm in prod (pinned, automated)

Use a chart (Bitnami, your own, or MongoDB Operator) but install via **Terraform `helm_release`**, **Argo CD**, or **Flux** — with `values-prod.yaml` in git.

- Pin chart version and container image
- Use a proper `StorageClass` (not Docker Desktop `hostpath`)
- Add backups, monitoring, resource limits, TLS

**Note:** Bitnami public images moved to `bitnamilegacy` with no updates — weak choice for new prod; prefer managed DB or your own pinned images.

### 3. Your own manifests / chart (like 11_helm/mongodb)

Store StatefulSet, Service, Secret, PVC templates in **your repo**.

- Full control, same git workflow as the app
- **You** own HA, backups, upgrades, and security

Good for learning and some internal environments; heavy for most app teams running prod DBs on K8s.

---

## Comparison

| | Local (this repo) | Production |
|---|-------------------|------------|
| Install | `helm install` by hand | Terraform / GitOps |
| Database | Pod + PVC on cluster | Managed service or platform-run StatefulSet |
| Storage | `hostpath`, single node | Cloud disk, replicated storage |
| Data on uninstall | PVC often kept | Backups + retention policy |
| Bitnami | OK for practice | Pin versions or avoid |

---

## Practical recommendation

1. **Prod:** managed MongoDB + app env vars from Secrets  
2. **Staging:** same as prod, or Helm with pinned versions  
3. **Local:** keep using `11_helm` or `12_helm_mongo` as you do now  

“Own files” in prod means **your values, secrets, and pipeline in git** — not necessarily avoiding Helm. It means not depending on an unpinned third-party chart installed manually.
