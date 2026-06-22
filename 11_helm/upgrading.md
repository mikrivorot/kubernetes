## Upgrading

`helm install` creates a new release. After that, use **`helm upgrade`** to apply changes to an existing release.

You need upgrading when something in the chart or its configuration changes and you want the cluster to match:

- **Scaling** — change replica count, resource limits, or probe settings
- **New image version** — update `image.tag` in `values.yaml` or via `--set`
- **Config or secrets** — change values in `values.yaml`; with our checksum annotations, this also triggers a pod rollout (see [configmap_rollout.md](./configmap_rollout.md))
- **Chart template changes** — edits to files under `templates/` after the first install

`helm upgrade` renders the chart again, compares it to the current release, and applies only what changed. Helm also stores a new revision, so you can inspect history or roll back if needed.

Example — scale the API to 5 replicas:

```bash
helm upgrade grade-submission-api ./grade-submission-api -n grade-submission --set replicaCount=5
```

For first-time deploy or idempotent scripts, use `helm upgrade --install` instead — it installs if the release does not exist, or upgrades if it does.
