# Enterprise GitOps Platform — Helm

> **Part of a 3-repository GitOps system.**
> This repo is the GitOps target — ArgoCD watches it and deploys whatever is here to EKS.
> You rarely touch this repo manually; the app repo's CI pipeline updates it automatically.
>
> | Repo | Purpose |
> |------|---------|
> | [`Enterprise-GitOps-Platform-app`](https://github.com/UsseF-1/Enterprise-GitOps-Platform-app) | Java source, Dockerfiles, CI pipeline |
> | [`Enterprise-GitOps-Platform-infra`](https://github.com/UsseF-1/Enterprise-GitOps-Platform-infra) | Terraform — EKS, ECR, VPC, SonarQube |
> | **`Enterprise-GitOps-Platform-helm`** ← you are here | Helm chart, ArgoCD manifests |

---

## What This Repo Does

This is the **GitOps source of truth** for the vprofile application stack on EKS.

- The **Helm chart** (`helm/vprofile/`) defines all Kubernetes resources for the app
- The **ArgoCD manifests** (`argoCD/`) tell ArgoCD to watch this repo and sync to EKS
- When the app repo's CI pipeline builds a new image, it commits a tag update to `helm/vprofile/values.yaml` in this repo
- ArgoCD detects the commit, renders the Helm chart, and deploys the change to EKS automatically — no manual `kubectl` needed

---

## Repository Structure

```
.
├── argoCD/
│   ├── app/
│   │   └── vprofile-app.yaml       # ArgoCD Application — points to helm/vprofile
│   └── projects/
│       └── vprofile-project.yaml   # ArgoCD AppProject — scopes access
├── helm/
│   └── vprofile/
│       ├── Chart.yaml              # Chart metadata
│       ├── values.yaml             # Default values (image tag updated by CI)
│       ├── output.yaml             # Rendered output for reference
│       └── templates/
│           ├── app-deployment.yaml      # vproapp Deployment + init containers
│           ├── db-deployment.yaml       # vprodb Deployment + PVC mount
│           ├── mc-deployment.yaml       # Memcached Deployment
│           ├── rmq-deployment.yaml      # RabbitMQ Deployment
│           ├── services.yaml            # ClusterIP services for all 4 components
│           ├── ingress.yaml             # AWS ALB Ingress (conditional)
│           ├── hpa.yaml                 # HorizontalPodAutoscaler for vproapp
│           ├── pvc.yaml                 # PersistentVolumeClaim for MySQL data
│           ├── secret.yaml              # Opaque secret for DB + RMQ passwords
│           └── dockerregistry-secret.yaml  # Optional private registry pull secret
└── k8s/                            # Raw manifests (reference only, not used by ArgoCD)
```

---

## Application Architecture on Kubernetes

```
                    Internet
                       │
              ┌────────▼────────┐
              │   AWS ALB       │  ← created by ALB Ingress Controller
              │  (port 80)      │
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  vproapp-service│  ClusterIP :8080
              └────────┬────────┘
                       │
         ┌─────────────▼─────────────┐
         │        vproapp            │  Deployment (HPA: 2–10 pods)
         │   Spring MVC / Tomcat     │
         └──┬──────────┬─────────────┘
            │          │
   ┌────────▼──┐  ┌────▼──────────┐
   │  vprodb   │  │  vprocache01  │
   │  MySQL    │  │  Memcached    │
   │  (+ PVC)  │  │               │
   └───────────┘  └───────────────┘
            │
   ┌────────▼──┐
   │  vpromq01 │
   │  RabbitMQ │
   └───────────┘
```

**Init containers** on `vproapp` ensure the pod only starts after `vprodb` and `vprocache01` DNS are resolvable.

---

## Deployed Resources

| Resource | Kind | Description |
|----------|------|-------------|
| `vproapp` | Deployment | Main application — image updated by CI |
| `vprodb` | Deployment | MySQL database |
| `vpromc` | Deployment | Memcached |
| `vpromq01` | Deployment | RabbitMQ |
| `vproapp-service` | Service (ClusterIP) | Routes to app on port 8080 |
| `vprodb` | Service (ClusterIP) | Routes to MySQL on port 3306 |
| `vprocache01` | Service (ClusterIP) | Routes to Memcached on port 11211 |
| `vpromq01` | Service (ClusterIP) | Routes to RabbitMQ on port 5672 |
| `vpro-ingress` | Ingress | AWS ALB — internet-facing, port 80 |
| `vproapp` | HPA | CPU-based autoscaling — min 2, max 10 pods |
| `db-pv-claim` | PVC | 8Gi EBS volume for MySQL data (gp2) |
| `<release>-app-secret` | Secret | DB password + RMQ password |

---

## ArgoCD Setup

### Prerequisites

- ArgoCD installed in the `argocd` namespace (see the infra repo for install steps)
- EKS cluster running and `kubectl` configured

### Apply the manifests

```bash
# Create the ArgoCD project first
kubectl apply -f argoCD/projects/vprofile-project.yaml

# Then create the Application
kubectl apply -f argoCD/app/vprofile-app.yaml
```

ArgoCD will immediately sync the Helm chart to the `vprofile` namespace (created automatically).

### Sync behavior

| Setting | Value |
|---------|-------|
| Auto-sync | ✅ enabled |
| Prune | ✅ enabled (removes resources deleted from Git) |
| Self-heal | ✅ enabled (reverts manual `kubectl` changes) |
| Namespace creation | ✅ automatic |

### Monitor sync status

```bash
# Via CLI
argocd app get vprofile
argocd app sync vprofile   # manual sync if needed

# Via UI — get the ArgoCD URL
kubectl get ingress -n argocd
```

---

## Helm Chart

### Install manually (without ArgoCD)

```bash
kubectl create namespace vprofile

helm install vprofile ./helm/vprofile \
  --namespace vprofile \
  --set ingress.host=yourdomain.com
```

### Upgrade

```bash
helm upgrade vprofile ./helm/vprofile --namespace vprofile
```

### Lint & dry-run

```bash
helm lint ./helm/vprofile
helm template vprofile ./helm/vprofile --debug
```

### Uninstall

```bash
helm uninstall vprofile --namespace vprofile
kubectl delete namespace vprofile
```

---

## Values Reference

Full defaults in `helm/vprofile/values.yaml`. Key values:

```yaml
app:
  image: <ECR_URL>/vprofile_app_image   # updated automatically by CI
  tag: <COMMIT_SHA>                      # updated automatically by CI
  replicas: 1
  resources:
    requests: { cpu: 200m, memory: 256Mi }
    limits:   { cpu: 500m, memory: 512Mi }
  hpa:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70

db:
  image: vprocontainers/vprofiledb
  tag: latest
  storageClass: gp2
  storageSize: 8Gi
  resources:
    requests: { cpu: 500m, memory: 1Gi }
    limits:   { cpu: "1",  memory: 2Gi }

ingress:
  enabled: true
  host: ""               # set to your domain
  certificateArn: ""     # set for HTTPS

secrets:
  dbPassword: vprodbpass  # change before production
  rmqPassword: guest      # change before production
```

### Override for production

```bash
helm upgrade vprofile ./helm/vprofile \
  --namespace vprofile \
  --set ingress.host=app.yourdomain.com \
  --set ingress.certificateArn=arn:aws:acm:us-east-1:ACCOUNT:certificate/CERT_ID \
  --set secrets.dbPassword=<STRONG_PASSWORD> \
  --set secrets.rmqPassword=<STRONG_PASSWORD>
```

---

## How the GitOps Loop Works

```
Developer pushes code to app repo
         │
         ▼
GitHub Actions (app repo ci.yml)
  1. Build Docker image
  2. Trivy scan
  3. Push to ECR with tag = short SHA
  4. git clone this repo (helm repo)
  5. yq update helm/vprofile/values.yaml
       app.image = <ECR_URL>
       app.tag   = <SHA>
  6. git commit + push
         │
         ▼
ArgoCD detects new commit in this repo
  1. Renders Helm chart with new values
  2. Compares with live EKS state
  3. Applies diff → rolling update to vproapp pods
         │
         ▼
New version live on EKS ✅
```

---

## Raw Kubernetes Manifests (`k8s/`)

The `k8s/` directory contains non-templated manifests covering the same stack. These are useful for:

- Understanding the raw resource structure before templating
- Quick manual testing on a local cluster (minikube/kind)
- Reference when debugging Helm-rendered output

They are **not** used by ArgoCD. To apply manually:

```bash
kubectl create namespace vprofile
kubectl apply -f k8s/ -n vprofile
```

---

## Troubleshooting

**App pods stuck in `Init` state**
Init containers wait for `vprodb` and `vprocache01` DNS. Check that the DB and Memcached deployments are healthy first:
```bash
kubectl get pods -n vprofile
kubectl describe pod <vproapp-pod> -n vprofile
```

**PVC stuck in `Pending`**
The EBS CSI driver must be running and the `gp2` StorageClass must exist:
```bash
kubectl get pods -n kube-system | grep ebs-csi
kubectl get storageclass
```

**ArgoCD OutOfSync after manual kubectl change**
Self-heal is enabled — ArgoCD will revert it within seconds. If you need to make a persistent change, update `values.yaml` in this repo instead.

**Image not updating**
Check that the CI pipeline's `update-helm` job completed successfully in the app repo's Actions tab, and verify the commit landed in this repo's `main` branch.
