## Deploying All Charts

Deploy all three charts together:

```bash
helm install grade-submission-api ./grade-submission-api -n grade-submission --create-namespace
helm install grade-submission-portal ./grade-submission-portal -n grade-submission
helm install mongodb ./mongodb -n grade-submission
```

### Does deploying make pods available?

**Partly — with a few layers.**

`helm install` does not create pods directly. It creates Kubernetes resources; controllers then create and run the pods.

| Chart | Workload resource | What creates pods |
|-------|-------------------|-------------------|
| `grade-submission-api` | Deployment | Deployment controller |
| `grade-submission-portal` | Deployment | Deployment controller |
| `mongodb` | StatefulSet | StatefulSet controller |

After a successful install, pods should appear once images are pulled and containers start:

```bash
kubectl get pods -n grade-submission
```

### What "available" means

1. **Pods exist and run** — containers start after `helm install`; this can take a minute.
2. **Pods are ready to serve traffic** — a pod can be `Running` but not `Ready` until readiness probes pass (the API chart configures these).
3. **Reachable inside the cluster** — each chart also creates a **Service** (ClusterIP). Other pods talk to each other via service names like `grade-submission-api` and `mongodb`.
4. **Reachable from your browser** — deploying these charts alone is not enough. The portal chart can create an **Ingress**, but you still need an ingress controller installed separately (see [traefik_ingress.md](./traefik_ingress.md)).

```
helm install  →  Deployment/StatefulSet + Service (+ Ingress for portal)
              →  Kubernetes schedules pods
              →  Service routes traffic to ready pods
              →  Ingress (if controller exists) exposes portal externally
```

Deploying the charts starts the process that makes pods available. The Deployment/StatefulSet creates pods; the Service makes them reachable in-cluster; Ingress + controller makes the portal reachable from outside.
