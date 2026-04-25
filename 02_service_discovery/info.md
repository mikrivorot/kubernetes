## Service

The Service primitive in Kubernetes abstracts away network complexities and provides a durable endpoint for accessing pods.

## NodePort

Allows external access to the Kubernetes network by exposing a static port on the node (range: 30000-32767).

Exposes the Service on a static port on **every Node** in the cluster.
External traffic reaches the app via `<NodeIP>:<nodePort>`.

- The Service also gets a cluster-internal IP, so it's reachable both externally and internally.
- Use case: development, quick demos, or when you don't have a cloud load balancer.
- Traffic flow: `External client → NodeIP:nodePort → Service → Pod:targetPort`

```yaml
spec:
  type: NodePort
  ports:
    - port: 5001        # internal cluster port
      targetPort: 5001  # port the container listens on
      nodePort: 32000   # externally accessible port on each Node
```

![alt text](image.png)

![alt text](image-1.png)
## ClusterIP

Used for internal pod-to-pod communication within the cluster.

Exposes the Service on an **internal IP only** — reachable solely within the cluster.
This is the default Service type if `type` is omitted.

- No external access. Pods talk to each other using the Service name as DNS (e.g. `grade-submission-api:8080`).
- Use case: internal communication between microservices (e.g. portal → API).
- Traffic flow: `Pod → ClusterIP:port → Service → Pod:targetPort`

```yaml
spec:
  type: ClusterIP       # or omit — this is the default
  ports:
    - port: 8080        # internal cluster port other Pods call
      targetPort: 8080  # port the container listens on
```

![alt text](image-2.png)

## Call Flow Diagram

```mermaid
flowchart TD
    Browser["Browser\nexternal client"]

    subgraph Node["Kubernetes Node"]
        subgraph PortalService["Service: grade-submission-portal\ntype: NodePort"]
            NP["nodePort: 32000\n→ port: 5001\n→ targetPort: 5001"]
        end

        subgraph PortalPod["Pod: grade-submission-portal\nlabel: instance=grade-submission-portal"]
            PC["container: grade-submission-portal\nlistens on :5001\nenv: GRADE_SUBMISSION_HOST=grade-submission-api"]
        end

        subgraph APIService["Service: grade-submission-api\ntype: ClusterIP (default)"]
            CP["port: 3000\n→ targetPort: 3000"]
        end

        subgraph APIPod["Pod: grade-submission-api\nlabel: instance=grade-submission-api"]
            AC["container: grade-submission-api\nlistens on :3000"]
        end
    end

    Browser -->|"HTTP :32000"| NP
    NP -->|"forwards to :5001"| PC
    PC -->|"grade-submission-api:3000\nvia DNS"| CP
    CP -->|"forwards to :3000"| AC
```

## Commands

```bash
# Apply all resources in the folder at once
kubectl apply -f service_discovery/

# Verify pods and services are running
kubectl get pods,services
```

output:

![alt text](image-3.png)



In app:

![alt text](image-4.png)


To shutdown
![alt text](image-5.png)