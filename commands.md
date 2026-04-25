```bash
kubectl config get-contexts
kubectl config use-context docker-desktop
```

---

```bash
# create or update any resource from a YAML file (Pod, Deployment, Service, ConfigMap, Namespace, etc.)
kubectl apply -f <file>
kubectl apply -f section_one/grade-submission-portal-pod.yaml

# delete and recreate if already exists
kubectl delete pod grade-submission-portal && kubectl apply -f section_one/grade-submission-portal-pod.yaml
```

```bash
kubectl describe pod grade-submission-portal
kubectl describe <type> <name>
kubectl logs grade-submission-portal
kubectl get pods

# stream logs
kubectl logs -f <pod> -c <container name, optional>
kubectl logs -f grade-submission-portal
kubectl logs -f grade-submission-portal -c grade-submission-portal-health-checker
```

```bash
# forward port to access container app locally
kubectl port-forward grade-submission-portal <local-port>:<pod-port>
```

```bash
# delete all pods with a specific label
kubectl delete pods -l "app.kubernetes.io/name=grade-submission"
```

```bash
# reapply deployment changes (use instead of deleting pods — controller will recreate them anyway)
kubectl apply -f grade-submission-api-deployment.yaml
kubectl apply -f grade-submission-portal-deployment.yaml
```

```bash
# list all Services in a namespace — shows type, ports, NodePort
kubectl get svc -n grade-submission
```

```bash
# show memory and CPU usage per pod (requires Metrics Server)
kubectl top pods -n grade-submission
```
