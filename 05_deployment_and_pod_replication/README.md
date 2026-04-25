

If we define a Deployment, we can specify the number of replicas we want to run and Kubertenes will create the specified number of Pods with identical appication running on them.

If we delete a Pod, Kubernetes will automatically create a new one.

![alt text](image.png)


A **cluster** is the set of machines (nodes) that Kubernetes manages — one control plane node running the Kubernetes brain, and worker nodes running your application pods.

Deployment and ReplicaSet are **Kubernetes objects** — defined in YAML files and submitted to the cluster via `kubectl apply`. The YAML file is just the definition; the object is what lives and runs in the cluster.

## Deployment

A Kubernetes object defined in YAML and submitted to the cluster via `kubectl apply`. A high-level controller that manages ReplicaSets and enables declarative updates (rolling updates, rollbacks). When you update a Deployment (e.g. new image), it creates a new ReplicaSet and gradually shifts traffic to it, keeping the old one for rollback.

A Deployment controller is a control loop running in the control plane that continuously watches the cluster and reconciles actual state to desired state — creating or updating ReplicaSets as needed. "Deployment" and "Deployment controller" refer to the same object; "controller" just emphasizes its behavioral role.

## ReplicaSet

Ensures a specified number of identical pod replicas are running at all times.

A ReplicaSet controller is a control loop running in the control plane that continuously watches the cluster and reconciles actual state to desired state. ReplicaSet watches how many pods matching its label selector are running — if the count doesn't match `replicas`, it creates or deletes pods to compensate.

You rarely create ReplicaSets directly — a Deployment creates and manages them for you.

![alt text](image-1.png)

## Reapplying changes

Deleting pods won't work — the Deployment controller will immediately recreate them. To apply changes to a Deployment, reapply the YAML:

```bash
kubectl apply -f grade-submission-api-deployment.yaml
kubectl apply -f grade-submission-portal-deployment.yaml
```

Kubernetes will perform a rolling update — gradually replacing old pods with new ones, without downtime.

Before modifications:

![alt text](image-3.png)


Modifications application:
![alt text](image-2.png)


Applied: 

![alt text](image-4.png)

## Inspecting Services

Lists all Services in a namespace — shows type, cluster IP, ports, and NodePort:

```bash
kubectl get svc -n grade-submission
```


![alt text](image-5.png)


## Stopping the cluster

**Delete everything** (namespace + all objects inside it):
```bash
kubectl delete namespace grade-submission
```

**Scale down to 0** (keeps objects, stops all pods — no need to touch Services, they're just routing rules with no pods to route to):
```bash
kubectl scale deployment grade-submission-portal -n grade-submission --replicas=0
kubectl scale deployment grade-submission-api -n grade-submission --replicas=0
```

**Scale back up:**
```bash
kubectl scale deployment grade-submission-portal -n grade-submission --replicas=1
kubectl scale deployment grade-submission-api -n grade-submission --replicas=3
```

## Deployment Flow

```
Browser
  │
  │ http://localhost:32000
  ▼
┌─────────────────────────────────────────────┐
│  Service: grade-submission-portal (NodePort)│
│  nodePort: 32000 → port: 5001               │
└─────────────────┬───────────────────────────┘
                  │ selects by: instance=grade-submission-portal
                  ▼
┌─────────────────────────────────────────────┐
│  Deployment: grade-submission-portal        │
│  replicas: 1                                │
│  └── Pod: grade-submission-portal           │
│        containerPort: 5001                  │
│        env: GRADE_SERVICE_HOST=             │
│             grade-submission-api            │
└─────────────────┬───────────────────────────┘
                  │ calls service by name: grade-submission-api:3000
                  ▼
┌─────────────────────────────────────────────┐
│  Service: grade-submission-api (ClusterIP)  │
│  port: 3000 → targetPort: 3000              │
└─────────────────┬───────────────────────────┘
                  │ selects by: instance=grade-submission-api
                  ▼
┌─────────────────────────────────────────────┐
│  Deployment: grade-submission-api           │
│  replicas: 3                                │
│  ├── Pod 1: grade-submission-api            │
│  ├── Pod 2: grade-submission-api            │
│  └── Pod 3: grade-submission-api            │
│        containerPort: 5001                  │
└─────────────────────────────────────────────┘
```