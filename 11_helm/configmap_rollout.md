## ConfigMap updates and rollout behavior

**If you change a ConfigMap used by a Deployment, Kubernetes does not restart existing pods automatically. The pod only receives updated environment variables when it starts.**

We force a rollout by adding a checksum annotation to the pod template metadata. When the rendered Helm template changes, the Deployment spec changes, and Kubernetes creates new pods:

- Helm renders templates and computes the final manifest.
- `helm upgrade` compares the new manifest against the current release.
- A changed pod template annotation causes the Deployment controller to perform a rolling update.
- New pods start with the updated ConfigMap values.

### Where this is implemented

- [grade-submission-api/templates/deployment.yaml](./grade-submission-api/templates/deployment.yaml) — `checksum/config` and `checksum/secret`
- [grade-submission-portal/templates/deployment.yaml](./grade-submission-portal/templates/deployment.yaml) — `checksum/config`

### Is this a best practice?

**Yes — for this setup it is a common and recommended pattern.**

When a Deployment loads ConfigMaps or Secrets via `envFrom` (as both charts do), Kubernetes does not reload env vars in running pods. A checksum annotation on the pod template is a lightweight way to tie pod restarts to config changes during `helm upgrade`, without extra tooling.

**Trade-offs to keep in mind:**

- The hash is computed from `values.yaml` data (`toJson .Values.config` / `.Values.secrets`), not from the live ConfigMap object in the cluster. That matches how these charts are structured and is fine for install/upgrade workflows.
- Any change to those values triggers a rolling update, even if the effective config is unchanged in edge cases (for example, key reordering in YAML).
- Alternatives exist: manual `kubectl rollout restart`, operators like [Reloader](https://github.com/stakater/Reloader), or mounting config as files instead of env vars (each has different trade-offs).

Learn more:
- Kubernetes ConfigMap docs: https://kubernetes.io/docs/concepts/configuration/configmap/
- Kubernetes Deployment rollout docs: https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- Helm upgrade docs: https://helm.sh/docs/helm/helm_upgrade/
- Helm template functions: https://helm.sh/docs/chart_template_guide/functions_and_pipelines/
