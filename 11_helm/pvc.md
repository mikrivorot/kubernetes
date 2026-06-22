## PersistentVolumeClaims (PVC)

### Why a PVC can remain after `helm uninstall`

After uninstalling the MongoDB chart, you may still see a PVC like:

```text
persistentvolumeclaim/mongodb-persistent-storage-mongodb-0   Bound   ...   1Gi   27d
```

This is **expected**. `helm uninstall` removed the StatefulSet and pods, but **did not delete the PVC**.

### How MongoDB storage is created

The MongoDB chart uses a **StatefulSet** with **`volumeClaimTemplates`** — see [mongodb/templates/statefulset.yaml](./mongodb/templates/statefulset.yaml):

```yaml
volumeClaimTemplates:
- metadata:
    name: mongodb-persistent-storage
  spec:
    accessModes: ...
    resources:
      requests:
        storage: ...
```

What that means:

| Who creates it | What |
|----------------|------|
| Helm | StatefulSet manifest (via chart template) |
| Kubernetes StatefulSet controller | One PVC per pod: `mongodb-persistent-storage-mongodb-0`, `mongodb-persistent-storage-mongodb-1`, … |

Helm never renders a standalone `PersistentVolumeClaim` YAML file. The PVC is a **child resource** created by Kubernetes when the StatefulSet runs.

### Why Kubernetes keeps PVCs

When a StatefulSet is deleted, Kubernetes **does not automatically delete** PVCs from `volumeClaimTemplates`. This is intentional — it protects database data from accidental loss when you tear down the workload.

`helm uninstall mongodb` deletes resources Helm owns (StatefulSet, Service, Secret, etc.). The PVC outlives the StatefulSet by design.

### Check what is left

```bash
kubectl get pods,pvc,deployments,statefulsets -n grade-submission
helm list -n grade-submission
```

After a full app uninstall you might see only the PVC — no pods, no deployments, no Helm releases in that namespace.

### Delete PVCs manually

When you want to free storage and remove leftover data:

```bash
# delete one PVC by name
kubectl delete pvc mongodb-persistent-storage-mongodb-0 -n grade-submission

# delete all PVCs in the namespace
kubectl delete pvc --all -n grade-submission
```

Full teardown including workloads and storage:

```bash
kubectl delete deployments,statefulsets,pvc --all -n grade-submission
```

### Reinstalling MongoDB

If you `helm install mongodb` again **without** deleting the PVC:

- The new StatefulSet reuses the existing PVC (same name pattern).
- **Old MongoDB data may still be on disk.**

For a fresh empty database, delete the PVC first, then reinstall:

```bash
helm uninstall mongodb -n grade-submission
kubectl delete pvc mongodb-persistent-storage-mongodb-0 -n grade-submission
helm install mongodb ./mongodb -n grade-submission
```

### Summary

| Action | Pods / StatefulSet | PVC |
|--------|-------------------|-----|
| `helm uninstall mongodb` | Removed | **Kept** |
| `kubectl delete pvc ...` | — | Removed |
| Reinstall without deleting PVC | Recreated | Reused (data may persist) |

See also: [uninstalling.md](./uninstalling.md)
