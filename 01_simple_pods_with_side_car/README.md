Kubernetes is a container orchestration platform that coordinates the collaboration of Master Nodes and Worker Nodes. Master Nodes (Control Plane) are responsible for scheduling and deciding where applications run. Worker Nodes provide the infrastructure to actually run the applications. In a single-node cluster, your computer plays the role of Master and Worker Node.



### Containers

Containers run applications in isolation with their dependencies, making them highly portable.

### Pods
In Kubernetes, pods are the smallest deployable units and encapsulate application containers.

Pods can run multiple containers, enabling sidecar patterns where auxiliary sidecar containers can communicate with the main application container via localhost.

Port forwarding creates a temporary connection between your local machine and a pod in the cluster. It's primarily used for debugging and testing purposes.

Every pod is given a virtual IP address, which is used to communicate with other pods in the cluster.

### Metadata vs Spec

**`metadata`** — identifies the object so other objects and tools can find and reference it (via name and labels).

**`spec`** — defines what the object should do or how it should behave.



