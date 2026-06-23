# Helm commands

Quick reference for Helm. Chart-specific docs live in [11_helm/](./11_helm/).

For raw Kubernetes operations (logs, scale, port-forward), see [k8s_commands.md](./k8s_commands.md).

---

## Repositories

Add a remote chart repo, refresh the index, and search charts.

```bash
# register a repo under a short name
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add traefik https://traefik.github.io/charts

# refresh chart indexes from all added repos
helm repo update

# search charts in a repo
helm search repo bitnami
```

See also: [11_helm/helm_collections.md](./11_helm/helm_collections.md)

---

## Install — grade submission charts

Deploy all three local charts from the `11_helm/` directory:

```bash
helm install grade-submission-api ./11_helm/grade-submission-api -n grade-submission --create-namespace
helm install grade-submission-portal ./11_helm/grade-submission-portal -n grade-submission
helm install mongodb ./11_helm/mongodb -n grade-submission
```

or 
(without mongo, for mongo we will use bitmani + command in a corresponding file)

```
cd ./grade-submission-api
helm install grade-submission-api . -n grade-submission

cd ../grade-submission-portal
helm install grade-submission-portal . -n grade-submission
```

```
grade-submission-api    grade-submission        1               2026-06-22 15:46:57.745815 +0300 +03    deployed        grade-submission-api-1.0.0
grade-submission-portal grade-submission        1               2026-06-22 15:47:17.590054 +0300 +03    deployed        grade-submission-portal-1.0.0 
```

Idempotent install or upgrade (recommended for scripts):

```bash
helm upgrade --install grade-submission-api ./11_helm/grade-submission-api -n grade-submission --create-namespace
helm upgrade --install grade-submission-portal ./11_helm/grade-submission-portal -n grade-submission --create-namespace
helm upgrade --install mongodb ./11_helm/mongodb -n grade-submission --create-namespace
```

Install from a remote repo:

```bash
helm install my-release bitnami/mysql
```

See also: [11_helm/chart_deployment.md](./11_helm/chart_deployment.md)

---

## Ingress controllers

### Traefik

```bash
helm repo add traefik https://traefik.github.io/charts && helm repo update && helm install traefik traefik/traefik --namespace traefik --create-namespace
```

Then deploy or upgrade the portal (needs Traefik for external access):

```bash
helm upgrade --install grade-submission-portal ./11_helm/grade-submission-portal -n grade-submission --create-namespace
```

See also: [11_helm/traefik_ingress.md](./11_helm/traefik_ingress.md)

---

## List and inspect releases

```bash
# releases in one namespace
helm list -n grade-submission

# all releases cluster-wide (-A = --all-namespaces)
helm list -A

# detailed info about one release
helm status grade-submission-api -n grade-submission

# revision history
helm history grade-submission-api -n grade-submission
```

See also: [11_helm/listing_deployments.md](./11_helm/listing_deployments.md), [11_helm/helm_collections.md](./11_helm/helm_collections.md)

---

## Upgrade

Apply changes to an existing release (scaling, image tag, config, template edits):

```bash
helm upgrade grade-submission-api ./11_helm/grade-submission-api -n grade-submission --set replicaCount=5
```

Install if missing, upgrade if present:

```bash
helm upgrade --install grade-submission-api ./11_helm/grade-submission-api -n grade-submission --create-namespace
```

Roll back to a previous revision:

```bash
helm rollback grade-submission-api 1 -n grade-submission
```

Preview rendered manifests without applying (dry run):

```bash
helm template grade-submission-api ./11_helm/grade-submission-api -n grade-submission
helm upgrade grade-submission-api ./11_helm/grade-submission-api -n grade-submission --dry-run
```

Config changes trigger pod rollouts via checksum annotations — see [11_helm/configmap_rollout.md](./11_helm/configmap_rollout.md).

See also: [11_helm/upgrading.md](./11_helm/upgrading.md)

---

## Package a chart

Create a `.tgz` archive for distribution or install from the package:

```bash
helm package ./11_helm/mongodb
helm install mongodb ./mongodb-0.1.0.tgz -n grade-submission --create-namespace
```

Note: `helm install ./11_helm/mongodb` installs from the chart directory directly and does not create a `.tgz`.

See also: [11_helm/chart_packaging.md](./11_helm/chart_packaging.md)

---

## Uninstall

Remove app releases (clears release records and deletes chart resources):

```bash
helm uninstall grade-submission-api -n grade-submission
helm uninstall grade-submission-portal -n grade-submission
helm uninstall mongodb -n grade-submission
```

Uninstall multiple releases in one namespace:

```bash
helm uninstall grade-submission-api grade-submission-portal mongodb -n grade-submission
```

Stop everything including ingress controllers:

```bash
helm uninstall grade-submission-api grade-submission-portal mongodb -n grade-submission
helm uninstall traefik -n traefik
helm uninstall ingress-nginx -n ingress-nginx
```

Verify nothing is left:

```bash
helm list -A
```

If you deleted Kubernetes objects manually but kept the release record, use `helm upgrade` to recreate resources — not `helm install`. See [11_helm/uninstalling.md](./11_helm/uninstalling.md).

MongoDB PVCs are kept after uninstall — see [11_helm/pvc.md](./11_helm/pvc.md).

---

## Verify pods after deploy

Helm creates Deployments/StatefulSets; kubectl confirms pods are running:

```bash
kubectl get pods -n grade-submission
kubectl get pods -A
```
