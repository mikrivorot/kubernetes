```
kubectl config get-contexts
kubectl config use-context docker-desktop
```

---

`kubectl apply -f <file>` can create any Kubernetes resource type — whatever is defined in the YAML:

Pod
Deployment
Service
ConfigMap
Namespace
etc.

```
kubectl apply -f section_one/grade-submission-portal-pod.yaml

# if already exists
kubectl delete pod grade-submission-portal && kubectl apply -f section_one/grade-submission-portal-pod.yaml

# pod/grade-submission-portal deleted
# pod/grade-submission-portal created

```

```
kubectl describe pod grade-submission-portal
kubectl logs grade-submission-portal
kubectl get pods

# stream logs
kubectl logs -f grade-submission-portal

```

```
# forwar port to access containter app
kubectl port-forward grade-submission-portal <local machine>:<port fron yaml>
```



Use label selector to delete all resources with a specific label
```
kubectl delete -l "app.kubernetes.io/name=grade-submission"
```


Pod can run multiple containers = created/terminated and scaled at the same time, same network, same port.
Can communicate with each other via localhost.

Example = side car pattern, health checker.