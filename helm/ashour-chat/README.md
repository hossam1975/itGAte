# ashour-chat Helm Chart

Deploys the Ashour Chat 3-tier app: **frontend** (nginx), **reactions** and **mood** (backend), **Redis** and **MySQL** (data layer), with Ingress and NetworkPolicies.

## Prerequisites

- Kubernetes cluster with an Ingress controller (e.g. NGINX)
- `kubectl` and `helm` 3.x
- TLS secret `ingress-tls` in `frontend` and `backend` namespaces (create manually or via pipeline secure files)

## Install

```bash
# From repo root
cd helm/ashour-chat

# Install with default values (images: frontend:latest, reactions:latest, mood:latest)
helm install ashour-chat . -n frontend --create-namespace

# Or use a release name and custom values (e.g. image prefix and tag from CI)
helm install ashour-chat . -f values.yaml --set global.imageRegistry=ashour11/ashour-chat --set frontend.image.tag=20260208.5 --set reactions.image.tag=20260208.5 --set mood.image.tag=20260208.5 -n frontend --create-namespace
```

The chart creates namespaces `frontend`, `backend`, and `data` and deploys all resources into them. The release is installed in the `frontend` namespace by convention; resources are created in all three namespaces.

## Upgrade

```bash
helm upgrade ashour-chat . -f values.yaml --set frontend.image.tag=<new-tag> --set reactions.image.tag=<new-tag> --set mood.image.tag=<new-tag> -n frontend
```

## Uninstall

```bash
helm uninstall ashour-chat -n frontend
# Namespaces and PVCs may remain; delete if desired:
# kubectl delete namespace frontend backend data --ignore-not-found --timeout=120s
```

## Values

| Key | Description | Default |
|-----|-------------|---------|
| `global.imageRegistry` | Image prefix (e.g. `ashour11/ashour-chat`). Images become `<prefix>-frontend:<tag>`, etc. | `""` |
| `ingress.enabled` | Create Ingress resources | `true` |
| `ingress.host` | Hostname for Ingress | `buzzboard.local` |
| `ingress.tls.secretName` | TLS secret name in frontend/backend | `ingress-tls` |
| `frontend.replicaCount` | Frontend replicas | `1` |
| `frontend.image.repository` / `tag` | Image (ignored if `global.imageRegistry` set) | `frontend:latest` |
| `reactions.replicaCount` | Reactions API replicas | `1` |
| `reactions.hpa.enabled` | Enable HPA for reactions | `true` |
| `mood.replicaCount` | Mood API replicas | `1` |
| `redis.replicaCount` | Redis replicas | `1` |
| `mysql.replicaCount` | MySQL replicas | `1` |
| `mysql.size` | MySQL PVC size | `2Gi` |
| `storageClass.name` | StorageClass for MySQL PVC | `standard-retain` |
| `storageClass.provisioner` | Cluster provisioner (e.g. `rancher.io/local-path` for k3d) | `rancher.io/local-path` |
| `secrets.*` | Redis/MySQL/JWT secrets (override in prod) | See `values.yaml` |

## Integration with pipeline

To deploy from the Azure pipeline using the chart instead of raw manifests:

1. Package the chart: `helm package helm/ashour-chat`
2. In the deploy job, run `helm upgrade --install ashour-chat ./ashour-chat-0.1.0.tgz ...` with `--set global.imageRegistry=$(DOCKERHUB_IMAGE_PREFIX)` and `--set frontend.image.tag=$(effectiveTag)` (and same for reactions/mood), or use a values file generated from pipeline variables.
