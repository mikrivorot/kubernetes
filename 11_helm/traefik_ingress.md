## Traefik Ingress Controller

Need Traefik ingress controller in your cluster first:

```bash
helm repo add traefik https://traefik.github.io/charts && helm repo update && helm install traefik traefik/traefik --namespace traefik --create-namespace
```

Then deploy or upgrade the portal chart:

```bash
helm upgrade --install grade-submission-portal ./grade-submission-portal -n grade-submission --create-namespace
```

After the controller is running, the portal should be reachable at `http://localhost/` if your local cluster supports host-based ingress or if Traefik is configured for local host routing.
