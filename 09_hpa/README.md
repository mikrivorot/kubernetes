## HPA

HPA stands for Horizontal Pod Autoscaler. It is a Kubernetes feature that automatically scales the number of pods in a deployment or statefulset based on CPU utilization (and only on CPU utilization, not on memory).

## Metrics Server

HPA relies on the metrics-server to read CPU/memory usage from kubelets. Install and configure it with:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml && kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":"--kubelet-insecure-tls"}]'
```

The first part installs the official metrics-server manifest (Deployment, Service, RBAC) in the `kube-system` namespace. The second part patches in the `--kubelet-insecure-tls` flag — required on local clusters (Docker Desktop, minikube) because kubelets there use self-signed certificates that metrics-server would otherwise reject.

Check pod CPU usage once metrics-server is running:

```bash
kubectl top pods -n grade-submission
```

![alt text](image.png)


![alt text](image-1.png)