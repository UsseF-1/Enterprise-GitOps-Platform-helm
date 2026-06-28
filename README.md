# Enterprise GitOps Platform Helm

## Overview

This repository contains a Helm chart and ArgoCD manifests for deploying the `vprofile` application stack to Kubernetes.

The project is designed for GitOps-style delivery and includes:

- A Helm chart in `helm/vprofile`
- ArgoCD `Application` and `AppProject` manifests in `argoCD/`
- Standalone Kubernetes manifest examples in `k8s/`

The application stack includes:

- `vproapp`: main application deployment
- `vprodb`: MySQL-compatible database deployment
- `vprocache01`: Memcached deployment
- `vpromq01`: RabbitMQ deployment
- Ingress support for HTTP traffic
- Horizontal Pod Autoscaling for the application
- Persistent storage for the database
- Kubernetes secrets for database and RabbitMQ credentials
- Optional private Docker registry image pull secret

---

## Repository Structure

- `argoCD/app/vprofile-app.yaml` - ArgoCD `Application` manifest for the Helm chart
- `argoCD/projects/vprofile-project.yaml` - ArgoCD project definition
- `helm/vprofile/Chart.yaml` - Helm chart metadata
- `helm/vprofile/values.yaml` - Default Helm values for the chart
- `helm/vprofile/templates/` - Kubernetes manifests rendered by Helm
- `k8s/` - Raw Kubernetes manifests for manual deployment or reference

---

## Requirements

- Kubernetes cluster (1.19+ recommended)
- Helm 3
- ArgoCD installed and configured in your cluster
- `kubectl` configured for the target cluster
- Optional: AWS ALB ingress controller when using the provided ALB annotations

---

## Helm Chart

The Helm chart is located at `helm/vprofile`.

### Install using Helm

```bash
kubectl create namespace vprofile
helm install vprofile ./helm/vprofile --namespace vprofile
```

### Upgrade

```bash
helm upgrade vprofile ./helm/vprofile --namespace vprofile
```

### Uninstall

```bash
helm uninstall vprofile --namespace vprofile
kubectl delete namespace vprofile
```

### Override values

You can customize the deployment by providing a custom values file or using `--set`.

```bash
helm install vprofile ./helm/vprofile --namespace vprofile --create-namespace \
  -f custom-values.yaml
```

Or:

```bash
helm install vprofile ./helm/vprofile --namespace vprofile --create-namespace \
  --set app.replicas=2 --set ingress.host=example.com
```

---

## ArgoCD Deployment

ArgoCD is configured to deploy the Helm chart directly from this repository.

### Apply ArgoCD manifests

```bash
kubectl apply -f argoCD/projects/vprofile-project.yaml
kubectl apply -f argoCD/app/vprofile-app.yaml
```

### How it works

- `argoCD/projects/vprofile-project.yaml` defines an ArgoCD `AppProject` that allows deployments from this Git repository to the `vprofile` namespace.
- `argoCD/app/vprofile-app.yaml` defines an ArgoCD `Application` that points to the Helm chart under `helm/vprofile` and uses `values.yaml`.
- The application is configured with automated sync, prune, and self-heal.
- ArgoCD will create the namespace automatically via `CreateNamespace=true`.

---

## Chart Values

The default chart values are defined in `helm/vprofile/values.yaml`.

Key sections include:

- `app` - main application image, runtime ports, replica count, resources, HPA settings, and default user
- `db` - database image, ports, storage settings, resources, and credentials
- `memcached` - memcached image, replicas, ports, and resources
- `rabbitmq` - RabbitMQ image, ports, user, and resources
- `initcontainers` - image used by init containers that wait for dependent services
- `ingress` - enable/disable ingress, hostname, port, and certificate ARN
- `dockerregistry` - optional private registry configuration
- `secrets` - database and RabbitMQ passwords used to generate Kubernetes secrets

### Default values highlights

- `app.replicas: 1`
- `app.hpa.enabled: true`
- `app.hpa.minReplicas: 2`
- `app.hpa.maxReplicas: 10`
- `db.storageSize: 8Gi`
- `ingress.enabled: true`
- `ingress.host: ""`
- `dockerregistry.enabled: false`

---

## Templates and Resources

The Helm chart renders the following Kubernetes resources:

- `Deployment` for `vproapp`
- `Deployment` for `vprodb`
- `Deployment` for `vprocache01`
- `Deployment` for `vpromq01`
- `Service` objects for application, database, cache, and RabbitMQ
- `Ingress` using ALB annotations when enabled
- `HorizontalPodAutoscaler` for the application when `app.hpa.enabled` is true
- `PersistentVolumeClaim` for database storage
- `Secret` for DB and RabbitMQ credentials
- Optional `dockerconfigjson` secret for private registries

### Important behavior

- `vproapp` uses init containers to wait for the database and memcached services to become resolvable.
- `vprodb` mounts a PVC named `db-pv-claim` and cleans `lost+found` during startup.
- `vpromq01` reads RabbitMQ password from the generated Kubernetes secret.

---

## Standalone Kubernetes Manifests

The `k8s/` folder contains raw manifests for manual deployment or reference.

Files include:

- `k8s/appdeploy.yaml`
- `k8s/appingress.yaml`
- `k8s/appservice.yaml`
- `k8s/dbdeploy.yaml`
- `k8s/dbpvc.yaml`
- `k8s/dbservice.yaml`
- `k8s/mcdep.yaml`
- `k8s/mcservice.yaml`
- `k8s/rmqdeploy.yaml`
- `k8s/rmqservice.yaml`
- `k8s/secret.yaml`

Use these manifests when you want a non-Helm or non-ArgoCD installation path.

---

## Configuration Notes

- Set `ingress.host` in `helm/vprofile/values.yaml` before deploying to enable routing.
- If using an AWS ALB ingress controller, ensure the cluster has the correct ALB controller installed.
- If you need to pull images from a private registry, enable `dockerregistry.enabled` and populate the registry fields.
- The secret values in `helm/vprofile/templates/secret.yaml` are base64-encoded at render time from `values.yaml`.

---

## Troubleshooting

- If the app pods remain pending, check PVC provisioning and storage class availability.
- If services cannot resolve, verify Kubernetes DNS and the namespace deployment target.
- If ArgoCD cannot sync, ensure the repository URL and branch/tag are correct in `argoCD/app/vprofile-app.yaml`.
- For ingress issues, verify the ingress controller and the `host` value.

---

## Customizing the Deployment

To customize resource requests, replica counts, or image tags, update `helm/vprofile/values.yaml` or pass overrides through Helm.

Example custom values file:

```yaml
app:
  replicas: 2
  tag: "61018f8"
  resources:
    requests:
      cpu: 300m
      memory: 512Mi
    limits:
      cpu: 1
      memory: 1Gi

ingress:
  enabled: true
  host: example.com

secrets:
  dbPassword: secure-db-password
  rmqPassword: secure-rmq-password
```

### Deploy with custom values

```bash
helm upgrade --install vprofile ./helm/vprofile --namespace vprofile --create-namespace -f custom-values.yaml
```

---

## Notes

- This repository is intended for a GitOps-style workflow using ArgoCD and Helm.
- The chart values are intentionally configurable for development and production environments.
- Replace placeholder values such as `ingress.host` and Docker image tags according to your environment.

---

## License

No license is specified in this repository. Add a `LICENSE` file if you want to define project licensing.
