# MLOps Platform Deployment Guide

## Architecture Overview

```
ArgoCD (GitOps Controller)
├── Ingress Controller (nginx-ingress)
│   ├── Namespace: ingress-nginx
│   ├── Service Type: LoadBalancer
│   └── Replicas: 2
│
├── MinIO (Object Storage)
│   ├── Namespace: minio
│   ├── Mode: Standalone
│   ├── Storage: 10Gi
│   └── Buckets: mlflow-artifacts, training-data, models
│
└── Kubeflow (ML Platform)
    ├── Namespace: kubeflow
    ├── Components: Jupyter, Katib, KFServing, Pipelines
    └── Integration: MinIO backend
```

## Prerequisites

- Kubernetes cluster 1.24+
- kubectl configured
- ArgoCD installed (`argocd` namespace)
- ~20Gi available storage
- Sufficient CPU/Memory resources

## Installation Steps

### Step 1: Verify ArgoCD Installation

```bash
kubectl get namespace argocd
kubectl get pods -n argocd
```

If ArgoCD is not installed:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 2: Create AppProject

```bash
kubectl apply -f infra/argocd/argocd-appproject.yaml

# Verify
kubectl get appproject -n argocd
```

### Step 3: Deploy Applications

Deploy in this order (they have no dependencies on each other):

```bash
# Deploy Ingress Controller
kubectl apply -f infra/argocd/ingress-nginx-app.yaml

# Deploy MinIO
kubectl apply -f infra/argocd/minio-app.yaml

# Deploy Kubeflow (takes longest, ~10-15 minutes)
kubectl apply -f infra/argocd/kubeflow-app.yaml
```

### Step 4: Monitor Deployment

```bash
# Watch application sync status
kubectl get applications -n argocd -w

# Detailed status for each app
kubectl describe app ingress-nginx -n argocd
kubectl describe app minio -n argocd
kubectl describe app kubeflow -n argocd

# Check pod status in each namespace
kubectl get pods -n ingress-nginx
kubectl get pods -n minio
kubectl get pods -n kubeflow
```

## Accessing Services

### Ingress Controller

```bash
# Get external IP
kubectl get svc -n ingress-nginx

# Example output:
# ingress-nginx-controller    LoadBalancer   10.0.1.1    35.1.1.1   80:31234/TCP,443:31456/TCP

# Use the EXTERNAL-IP for DNS configuration
```

### MinIO Console

```bash
# Option 1: Port Forward
kubectl port-forward -n minio svc/minio 9001:9001
# Access at http://localhost:9001
# Login: minioadmin / minioadmin

# Option 2: Expose via Ingress (requires DNS)
# Create Ingress resource pointing to minio service
```

### Kubeflow Dashboard

```bash
# Option 1: Port Forward
kubectl port-forward -n kubeflow svc/centraldashboard 8080:80
# Access at http://localhost:8080

# Option 2: Via Ingress (requires Kubeflow auth setup)
```

## Configuration Customization

### Change MinIO Credentials

Edit `infra/helm/values-minio.yaml`:

```yaml
auth:
  rootUser: your-custom-user
  rootPassword: your-secure-password
```

Then apply:

```bash
kubectl apply -f infra/argocd/minio-app.yaml
```

### Adjust Storage Size

Edit MinIO values:

```yaml
persistence:
  size: 50Gi  # Change as needed
```

Or Ingress resources:

```yaml
controller:
  replicaCount: 3  # Increase replicas
  resources:
    limits:
      memory: "1Gi"  # Increase limits
```

### Enable TLS for Ingress

Create a certificate secret and update Ingress rules (additional configuration needed).

## Troubleshooting

### Application Not Syncing

```bash
# Force sync
argocd app sync ingress-nginx --server <argocd-server>

# Or via kubectl
kubectl patch app ingress-nginx -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":{"prune":true}}}}'
```

### Pod Startup Issues

```bash
# Check pod logs
kubectl logs -n minio -l app=minio --tail=100
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --tail=100

# Describe failed pod
kubectl describe pod <pod-name> -n <namespace>
```

### Storage Issues

```bash
# Check PVC status
kubectl get pvc -n minio
kubectl describe pvc <pvc-name> -n minio

# Verify storage class exists
kubectl get storageclass
```

### Kubeflow Installation Stuck

Kubeflow uses kustomize which can take 10-15 minutes. Check ArgoCD logs:

```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-controller-manager -f
```

## Monitoring Sync Status

### Via ArgoCD CLI

```bash
# Install argocd CLI if not present
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/v2.8.0/argocd-linux-amd64
chmod +x argocd-linux-amd64
sudo mv argocd-linux-amd64 /usr/local/bin/argocd

# Port forward to ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login
argocd login localhost:8080

# Check app status
argocd app get ingress-nginx
argocd app get minio
argocd app get kubeflow
```

### Via kubectl

```bash
# Watch all apps
watch kubectl get applications -n argocd

# JSON output for parsing
kubectl get app -n argocd -o json
```

## Next Steps

1. **Configure DNS**: Point domain to Ingress LoadBalancer IP
2. **Setup TLS**: Install cert-manager and create certificates
3. **Create Kubeflow Ingress**: Route traffic to dashboard
4. **Configure MinIO Buckets**: Create additional buckets as needed
5. **Setup Monitoring**: Install Prometheus/Grafana
6. **Create ML Pipelines**: Use Kubeflow Pipelines for workflows

## Production Checklist

- [ ] Change MinIO root password
- [ ] Setup persistent storage with backups
- [ ] Configure network policies
- [ ] Enable pod security policies
- [ ] Setup resource quotas per namespace
- [ ] Enable audit logging
- [ ] Configure monitoring and alerting
- [ ] Setup disaster recovery plan
- [ ] Document team access procedures
- [ ] Setup image registry integration

## Cleanup

To remove all deployed components:

```bash
# Delete applications (ArgoCD will auto-cleanup resources)
kubectl delete app ingress-nginx minio kubeflow -n argocd

# Wait for resource deletion
kubectl get all -n ingress-nginx
kubectl get all -n minio
kubectl get all -n kubeflow

# Optional: Delete namespaces
kubectl delete namespace ingress-nginx minio kubeflow
```

## Support and Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [MinIO Documentation](https://docs.min.io/)
- [Kubeflow Documentation](https://www.kubeflow.org/docs/)
- [nginx-ingress Helm Chart](https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx)