# Assessment App — GitOps Repository

This repository contains the Kubernetes deployment manifests and Helm chart for the Assessment App, managed by ArgoCD.

## Structure

```
devops-assessment-gitops/
├── base/                        ← Base K8s manifests (Kustomize)
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── httproute.yaml           ← Gateway API (not Ingress)
│   └── kustomization.yaml
│
├── overlays/
│   └── prod/                    ← Prod: 3 replicas, assessment.jaali.dev
│       └── kustomization.yaml
│
├── argocd/
│   └── application-prod.yaml    ← Auto-sync enabled
│
├── chart/                       ← Helm chart (can be packaged to Harbor OCI)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       ├── service.yaml
│       └── httproute.yaml
│
└── README.md
```

## How It Works

1. Jenkins CI builds and pushes `harbor.jaali.dev/assessment/assessment-app:prod-<commit-sha>` to Harbor
2. Jenkins updates `overlays/prod/kustomization.yaml` with the new tag via git commit
3. ArgoCD detects the change and syncs production automatically

## Image Tagging

Images are **never** tagged `latest` or `stable`. Every deployment uses an immutable tag:

```
prod-<8-char-commit-sha>   e.g. prod-a1b2c3d4
```

## Deployment

```bash
# Register the ArgoCD application (one-time setup)
kubectl apply -f argocd/application-prod.yaml
```

## Helm Chart Usage

The chart can be used standalone or packaged as an OCI artifact in Harbor:

```bash
# Local install
helm install assessment-app ./chart -n assessment-app-prod --create-namespace \
  --set image.tag="prod-abc12345" \
  --set replicaCount=3 \
  --set gateway.hostname="assessment.jaali.dev"

# Package and push to Harbor (OCI)
helm package ./chart
helm push assessment-app-0.1.0.tgz oci://harbor.jaali.dev/assessment
```

## Rollback

```bash
# Via ArgoCD (preferred)
argocd app history assessment-app-prod
argocd app rollback assessment-app-prod <revision>

# Via Git (revert the tag change)
git revert HEAD
git push origin main
# ArgoCD auto-syncs to the previous image
```
# devops-assessment-gitops
