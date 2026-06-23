# Grade submission + Bitnami MongoDB

Same app charts as [11_helm](../11_helm/), but MongoDB comes from the **Bitnami** chart instead of the custom `11_helm/mongodb` chart.

## Key difference from 11_helm

| | 11_helm | 12_helm_mongo |
|---|---------|---------------|
| MongoDB | Custom chart → Service named `mongodb` | Bitnami chart |
| API `MONGODB_HOST` | `mongodb` | `mongodb` (same) |
| API credentials | `MONGODB_USER` + `MONGODB_PASSWORD` (separate env vars) | same |

The API builds the connection string in `app.js` from separate env vars — not a full URI:

```
mongodb://${MONGODB_USER}:${MONGODB_PASSWORD}@${MONGODB_HOST}:${MONGODB_PORT}/
```

**Important:** Bitnami service name depends on the **Helm release name**. Use release name `mongodb` so the Service is `mongodb` — same DNS the API expects:

```
helm install mongodb ...   →  Service: mongodb   ✓  (matches MONGODB_HOST: mongodb)
helm install mongo ...     →  Service: mongo-mongodb   ✗
```

Install everything in **`grade-submission`** namespace (same as API and portal).

## Setup

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami/mongodb --versions
```

Export default values (run from `mongodb/`):

```bash
cd mongodb
helm show values bitnami/mongodb > default_values.yaml
```

If fetch fails:

```bash
helm cache clean
helm repo update bitnami
```

## Install MongoDB (Bitnami)

From `mongodb/` directory. See [values.yaml](./mongodb/values.yaml) for `bitnamilegacy` image and auth settings.

```bash
helm install mongodb bitnami/mongodb --version 15.6.13 -f values.yaml -n grade-submission --create-namespace
```

## Install app charts

From repo root or `12_helm_mongo/`:

```bash
helm install grade-submission-api ./grade-submission-api -n grade-submission
helm install grade-submission-portal ./grade-submission-portal -n grade-submission
```

Or upgrade if already installed (use `--reset-values` if old `--set` overrides linger):

```bash
helm upgrade grade-submission-api ./grade-submission-api -n grade-submission -f grade-submission-api/values.yaml --reset-values
```

## Chart vs container image

| In the command | Meaning | Example |
|----------------|---------|---------|
| `bitnami/mongodb` | Helm chart | templates for StatefulSet, Service, etc. |
| `--version 15.6.13` | Chart version | not the Docker tag |
| `-f values.yaml` | Overrides | image repo, auth, etc. |

Do **not** put `bitnamilegacy/mongodb` in the install command — that goes under `image:` in `values.yaml`.

## Troubleshooting

### ImagePullBackOff (`bitnami/mongodb:... not found`)

Override image to `bitnamilegacy` in `values.yaml`, then:

```bash
helm upgrade mongodb bitnami/mongodb --version 15.6.13 -f values.yaml -n grade-submission
kubectl delete pod mongodb-0 -n grade-submission
```

### API pods 0/1 Ready

1. Service must be named `mongodb` → release name must be `mongodb` in `grade-submission`
2. Auth must match API secrets (`admin` / `password123`) → `auth.enabled: true` in mongo `values.yaml`
3. If mongo was first installed without auth, delete PVC and reinstall:

```bash
helm uninstall mongodb -n grade-submission
kubectl delete pvc datadir-mongodb-0 -n grade-submission
helm install mongodb bitnami/mongodb --version 15.6.13 -f values.yaml -n grade-submission
```
