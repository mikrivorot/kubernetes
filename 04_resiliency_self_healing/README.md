# Resiliency & Self-Healing

By default, every container in a pod has a restart policy of "Always". This means Kubernetes restarts a container whenever its process terminates, regardless of the exit code.

Kubernetes automatically detects and restarts terminated containers

### Process Termination vs. Application Unresponsiveness


Process Termination:

Kubernetes automatically detects when a process within a container terminates.

Kubernetes immediately takes action based on the restart policy, without any additional configuration, as seen in the lesson.

Examples:

Out of Memory (OOM) Error: If a container exceeds its memory limit, the process is terminated by the system, triggering a restart.

Application Crash: If your application encounters an unhandled exception and exits, Kubernetes will restart the container.

Application Unresponsiveness:

This occurs when an application is still running but not functioning correctly.

Kubernetes cannot automatically detect this situation without us additionally configuring liveness probes, which will be covered in a future lesson.

Examples:

Deadlock Situations: In cases where your application becomes unresponsive due to a deadlock, Kubernetes can restart the container if configured with appropriate liveness probes

### restartPolicy vs pod recreation

`restartPolicy: Always` restarts **containers inside a pod** if they crash — it does not recreate the pod itself if it is deleted.

Bare pods (`kind: Pod`) have no guardian — once deleted, they are gone.

Pods managed by a **controller** (Deployment, ReplicaSet, StatefulSet) are recreated on deletion because the controller continuously reconciles the desired state ("I want N replicas") against the actual state. The only way to stop recreation is to downscale to 0 replicas.
