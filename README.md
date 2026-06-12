# Kustomize Multi-Environment Kubernetes

Production-ready Kustomize configuration demonstrating the base + overlays pattern for deploying microservices across 4 environments (dev, qa, staging, prod).

![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28+-326CE5?logo=kubernetes)
![Kustomize](https://img.shields.io/badge/Kustomize-5.0+-blue)
![License](https://img.shields.io/badge/License-MIT-green)

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    GitOps Pipeline                         │
│  Push → GitHub Actions → Kustomize Build → kubectl apply  │
└──────────────┬───────────────────────────────────────────┘
               │
    ┌──────────┼──────────┬──────────┬──────────┐
    ▼          ▼          ▼          ▼          
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│  DEV   │ │   QA   │ │STAGING │ │  PROD  │
│ 1 node │ │2 nodes │ │2 nodes │ │3 nodes │
│  B4ms  │ │ D4s_v5 │ │ D4s_v5 │ │ D8s_v5 │
└────────┘ └────────┘ └────────┘ └────────┘
```

## Directory Structure

```
.
├── base/                          # Shared base manifests
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── common/                    # Shared ConfigMaps, RBAC
│   │   ├── kustomization.yaml
│   │   └── configmap.yaml
│   ├── api-service/               # Microservice: API Gateway
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml
│   ├── auth-service/              # Microservice: Authentication
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── worker-service/            # Microservice: Background Jobs
│       ├── kustomization.yaml
│       ├── deployment.yaml
│       └── service.yaml
├── overlays/
│   ├── dev/                       # Dev overrides
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   └── patches/
│   │       ├── replicas.yaml
│   │       └── keyvault-dev.yaml
│   ├── qa/
│   ├── staging/
│   └── prod/
│       ├── kustomization.yaml
│       ├── namespace.yaml
│       └── patches/
│           ├── replicas.yaml
│           ├── resources.yaml
│           ├── keyvault-prod.yaml
│           └── ingress-prod.yaml
└── components/                    # Reusable Kustomize components
    ├── ingress-nginx/
    ├── csi-secrets-store/
    └── monitoring/
```

## How It Works

### Base Layer
All services define their core manifests (Deployment, Service, HPA) in `base/`. These are environment-agnostic.

### Overlay Layer
Each environment folder in `overlays/` applies patches to customize:
- **Replicas** — scale per environment
- **Resources** — CPU/memory limits per environment
- **Secrets** — Key Vault references per environment
- **Ingress** — domain names per environment

### Build & Deploy

```bash
# Preview what will be applied
kustomize build overlays/dev

# Apply to cluster
kubectl apply -k overlays/dev

# Or with kustomize CLI
kustomize build overlays/prod | kubectl apply -f -
```

## Environment Differences

| Setting | Dev | QA | Staging | Prod |
|---------|-----|----|---------|------|
| Replicas (API) | 1 | 2 | 2 | 3 |
| CPU Request | 100m | 250m | 250m | 500m |
| Memory Request | 128Mi | 256Mi | 256Mi | 512Mi |
| HPA Max | 2 | 4 | 4 | 10 |
| Key Vault | dev-kv | qa-kv | staging-kv | prod-kv |
| Ingress Host | *.dev.internal | *.qa.internal | *.staging.example.com | *.example.com |
| TLS | self-signed | self-signed | Let's Encrypt | Let's Encrypt |

## Key Patterns

### 1. SecretProviderClass per Environment
Each env patches the Key Vault name to pull secrets from the correct vault:
```yaml
# overlays/prod/patches/keyvault-prod.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kvname
spec:
  parameters:
    keyvaultName: "myplatform-prod-kv"
    tenantId: "your-tenant-id"
```

### 2. Strategic Merge Patches for Scaling
```yaml
# overlays/prod/patches/replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-service
spec:
  replicas: 3
```

### 3. Common Labels
Overlays add environment labels automatically:
```yaml
commonLabels:
  environment: prod
  app.kubernetes.io/managed-by: kustomize
```

## Prerequisites

- `kubectl` >= 1.28
- `kustomize` >= 5.0 (or built-in `kubectl -k`)
- AKS cluster with CSI Secrets Store driver installed
- NGINX Ingress Controller

## License

MIT
