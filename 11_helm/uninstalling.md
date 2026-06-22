## Uninstalling

### What is a release record?

A **release record** is Helm's metadata for one named chart installation in a namespace. It is stored in the cluster (usually as Secrets) and tracks the release name, namespace, chart version, revision history, and the manifests Helm applied.

It is separate from the live Kubernetes objects (Deployments, Services, etc.). Helm uses it to run `upgrade`, `rollback`, `history`, and `uninstall` against a known release.

If you delete only the Kubernetes objects, the release record still exists. That is why a new `helm install` with the same release name fails — Helm treats that name as already taken. Use `helm upgrade` instead to re-apply the chart and recreate the missing resources. To fully remove the release and start fresh, run `helm uninstall`.

```bash
helm uninstall grade-submission-api -n grade-submission
helm uninstall grade-submission-portal -n grade-submission
helm uninstall mongodb -n grade-submission
```
