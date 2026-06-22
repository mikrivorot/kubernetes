# Kubernetes (kubectl) commands

Quick reference for working with the cluster directly. For Helm-managed releases, see [helm_commands.md](./helm_commands.md).

## Context

```bash
kubectl config get-contexts
kubectl config use-context docker-desktop
```

## Apply and delete resources

```bash
# create or update any resource from a YAML file (Pod, Deployment, Service, ConfigMap, Namespace, etc.)
kubectl apply -f <file>
kubectl apply -f section_one/grade-submission-portal-pod.yaml

# delete and recreate if already exists
kubectl delete pod grade-submission-portal && kubectl apply -f section_one/grade-submission-portal-pod.yaml
```

```bash
# reapply deployment changes (use instead of deleting pods — controller will recreate them anyway)
kubectl apply -f grade-submission-api-deployment.yaml
kubectl apply -f grade-submission-portal-deployment.yaml
```

```bash
# delete all pods with a specific label
kubectl delete pods -l "app.kubernetes.io/name=grade-submission"
```

```bash
# delete all deployments in a namespace (cascades to ReplicaSets and Pods)
kubectl delete deployments --all -n <namespace>
kubectl delete deployments --all -n grade-submission

# delete deployments, statefulsets, and PVCs together (full teardown including storage)
kubectl delete deployments,statefulsets,pvc,hpa --all -n grade-submission
```

## Inspect pods and logs

```bash
kubectl describe pod grade-submission-portal
kubectl describe <type> <name>
kubectl logs grade-submission-portal
kubectl get pods

# list pods in all namespaces — cluster-wide view (status, restarts, age)
kubectl get pods -A

# stream logs
kubectl logs -f <pod> -c <container name, optional>
kubectl logs -f grade-submission-portal
kubectl logs -f grade-submission-portal -c grade-submission-portal-health-checker
```

## Services, deployments, and statefulsets

```bash
# list all Services in a namespace — shows type, ports, NodePort
kubectl get svc -n grade-submission
```

```bash
# view the full live spec of a deployment as Kubernetes sees it
kubectl get deployment <name> -n <namespace> -o yaml
kubectl get deployment grade-submission-portal -n grade-submission -o yaml
```

```bash
# list all StatefulSets in a namespace — shows desired vs ready replicas
kubectl get statefulset -n <namespace>
kubectl get statefulset -n grade-submission
# NAME      READY   AGE
# mongodb   2/2     30m

# show full details of a StatefulSet — events, pod template, volume claims
kubectl describe statefulset <name> -n <namespace>
kubectl describe statefulset mongodb -n grade-submission
```

## Storage

See [11_helm/pvc.md](./11_helm/pvc.md) for why PVCs survive `helm uninstall` and how to delete them.

```bash
# list all PersistentVolumeClaims in a namespace — shows status (Bound/Pending), PV name, size, and access mode
kubectl get pvc -n <namespace>
kubectl get pvc -n grade-submission
# NAME                                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# mongodb-persistent-storage-mongodb-0   Bound    pvc-8f2580b4-4385-4fc0-8476-8edffe16ff10   1Gi        RWO            hostpath       30m
# mongodb-persistent-storage-mongodb-1   Bound    pvc-f0213168-b460-410b-834e-7afdd50fe9a5   1Gi        RWO            hostpath       30m
```

## Scaling and metrics

```bash
# scale down all workloads to 0 (keeps objects, stops all pods)
kubectl scale deployment grade-submission-api grade-submission-portal -n grade-submission --replicas=0
# deployment.apps/grade-submission-api scaled
# deployment.apps/grade-submission-portal scaled
kubectl scale statefulset mongodb -n grade-submission --replicas=0
# statefulset.apps/mongodb scaled

# scale back up
kubectl scale deployment grade-submission-api grade-submission-portal -n grade-submission --replicas=1
kubectl scale statefulset mongodb -n grade-submission --replicas=2
```

```bash
# show memory and CPU usage per pod (requires Metrics Server)
kubectl top pods -n grade-submission
```

## Port forwarding

```bash
# forward port to access container app locally
kubectl port-forward grade-submission-portal <local-port>:<pod-port>
```

## Cluster health (after Helm deploy)

```bash
kubectl get nodes
kubectl get pods -A
kubectl get pods -n grade-submission
```
