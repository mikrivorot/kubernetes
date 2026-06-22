# Helm basics

## What `helm repo add` does

`helm repo add bitnami https://charts.bitnami.com/bitnami` tells Helm to register a remote chart repository under the name `bitnami`.

After that, Helm knows where to look when you want to install charts from that source.

## Why this is needed

A Helm repository is just a collection of packaged charts. Helm does not automatically know every possible chart source.

You must first add the repository so Helm can:
- list available charts,
- download chart metadata,
- install charts from that repo.

## Typical flow

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami
```

Example install:

```bash
helm install my-release bitnami/mysql
```

## What `helm repo update` does

`helm repo update` refreshes the local index of charts from all added repositories.

This ensures you see the newest available versions.

## What `helm list -A` does

`helm list` shows Helm releases in the current namespace only.

`helm list -A` lists releases across **all** namespaces.

The `-A` flag is short for `--all-namespaces`.

Example:

```bash
helm list -A
```

Typical output columns include release name, namespace, chart, app version, and status (for example `deployed` or `failed`).

## Reading `helm list -A` output

Each row is one Helm **release** — a named installation of a chart in your cluster.

| Column | Meaning |
|--------|---------|
| **NAME** | Release name you chose at install time (or the default). Used in other Helm commands, e.g. `helm upgrade grade-submission-api ...`. |
| **NAMESPACE** | Kubernetes namespace where that release lives. Resources from the chart are created there. |
| **REVISION** | How many times this release was upgraded. `1` = first install; higher numbers mean `helm upgrade` ran again. |
| **UPDATED** | When the current revision was last applied. |
| **STATUS** | Release state. `deployed` means the last install/upgrade succeeded. Other values include `failed`, `pending-install`, or `uninstalled`. |
| **CHART** | Chart name and version used for this release (format: `chart-name-version`). |
| **APP VERSION** | Application version baked into the chart (optional; some charts leave it blank). |

### Example from this cluster

```text
NAME                    NAMESPACE         REVISION  STATUS    CHART
grade-submission-api    grade-submission  2         deployed  grade-submission-api-1.0.0
grade-submission-portal grade-submission  4         deployed  grade-submission-portal-1.0.0
ingress-nginx           ingress-nginx     1         deployed  ingress-nginx-4.15.1
mongodb                 grade-submission  1         deployed  mongodb-1.0.0
traefik                 traefik           1         deployed  traefik-40.3.0
```

What this tells you:

- **Three namespaces** have Helm releases: `grade-submission`, `ingress-nginx`, and `traefik`.
- **`grade-submission`** holds the app stack: API, portal, and MongoDB. The API is on revision 2 and the portal on revision 4, so both were upgraded after the initial install.
- **`ingress-nginx`** and **`traefik`** are separate ingress controllers, each in its own namespace — common when different charts or teams manage routing.
- All releases show **`deployed`**, so the last operation on each one completed successfully.

## Does `deployed` mean the cluster is running?

Not entirely. **`deployed`** means Helm’s last install or upgrade for that release **succeeded**. It does not guarantee every pod is healthy right now.

**What `deployed` does tell you:**

- The cluster API is reachable (otherwise `helm list` would fail).
- Those releases exist and Helm applied their charts successfully.
- Kubernetes should have created the Deployments, Services, and other resources from those charts.

**What `deployed` does not guarantee:**

- Pods are still running (they can crash after a successful deploy).
- Apps respond correctly on the network.
- All nodes are healthy.

**Check whether things are actually running:**

```bash
kubectl get nodes
kubectl get pods -A
```

If nodes are `Ready` and pods are mostly `Running`, the cluster is up and your apps are likely running.

For one namespace:

```bash
kubectl get pods -n grade-submission
```

**Summary:** `deployed` = Helm finished successfully. Use `kubectl get pods -A` to confirm runtime state.

## Why `-A` is useful

By default, Helm scopes commands to the namespace in your current `kubectl` context.

If you installed charts in different namespaces, `helm list` without `-A` may show nothing or miss releases you expect to see.

Use `helm list -A` when you want a cluster-wide view of every Helm release.

## Simple analogy

Think of `helm repo add` like adding a bookmark or source list.

Think of `helm repo update` like refreshing that list.

Think of `helm install` like downloading and deploying something from that list.

Think of `helm list -A` like viewing everything you have installed from that list, no matter which room (namespace) it is in.
