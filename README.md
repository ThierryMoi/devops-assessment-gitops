# Assessment App — GitOps Repository

Kubernetes deployment manifests and Helm chart for the Assessment App, managed by ArgoCD on the [JaaLi platform](https://github.com/ThierryMoi/jaali-ai-platform).

## Structure

```
devops-assessment-gitops/
├── base/                        ← Shared Kustomize manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── httproute.yaml           ← Gateway API (HTTPRoute, not Ingress)
│   └── kustomization.yaml
│
├── overlays/
│   └── prod/                    ← Production overlay
│       ├── namespace.yaml       ← assessment-app-prod
│       └── kustomization.yaml   ← 3 replicas, prod-<sha> tag, resources
│
├── gateway/                     ← Cross-namespace Gateway API auth
│   ├── reference-grant.yaml     ← Lives in envoy-gateway-system
│   └── kustomization.yaml
│
├── argocd/
│   └── application-prod.yaml    ← ArgoCD Application (multi-source, auto-sync)
│
├── chart/                       ← Helm chart v0.2.0 (optional / Harbor OCI)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── namespace.yaml
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── httproute.yaml
│       └── reference-grant.yaml
│
└── README.md
```

## Platform Context (JaaLi)

| Component | Value |
|-----------|-------|
| Cluster Gateway | `jaali-gateway` in `envoy-gateway-system` |
| ArgoCD namespace | `ci-cd` (not `argocd`) |
| App namespace | `assessment-app-prod` |
| Hostname | `assessment.jaali.dev` |
| Image registry | `harbor.jaali.dev/assessment/assessment-app` |
| TLS | cert-manager + Let's Encrypt on `*.jaali.dev` |

The HTTPRoute in `assessment-app-prod` references `jaali-gateway` in another namespace. A **ReferenceGrant** in `gateway/` authorizes this cross-namespace attachment (same pattern as `platform`, `ci-cd`, `auth` in jaali-platform).

## How It Works

```
Jenkins CI  →  Harbor: assessment-app:prod-<sha>
            →  Git commit: overlays/prod/kustomization.yaml (newTag)
            →  ArgoCD auto-sync (multi-source)
                  ├── overlays/prod  → Deployment, Service, HTTPRoute, Namespace
                  └── gateway        → ReferenceGrant in envoy-gateway-system
```

1. Jenkins builds and pushes `harbor.jaali.dev/assessment/assessment-app:prod-<commit-sha>`
2. Jenkins updates `overlays/prod/kustomization.yaml` via git commit
3. ArgoCD detects the change and syncs production automatically

## Image Tagging

Images are **never** tagged `latest` or `stable`. Every deployment uses an immutable tag:

```
prod-<8-char-commit-sha>   e.g. prod-a1b2c3d4
```

## Prerequisites

- ArgoCD running in `ci-cd` namespace
- Envoy Gateway with `jaali-gateway` deployed
- Harbor project `assessment` with pull access from the cluster
- DNS `assessment.jaali.dev` pointing to the cluster load balancer
- GitHub repo registered in ArgoCD (private repo requires a PAT):

```bash
export KUBECONFIG=/path/to/jaali.yaml
export GITHUB_TOKEN=ghp_xxx

kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: repo-devops-assessment-gitops
  namespace: ci-cd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  name: devops-assessment-gitops
  type: git
  url: https://github.com/ThierryMoi/devops-assessment-gitops.git
  username: ThierryMoi
  password: ${GITHUB_TOKEN}
EOF
```

## Deployment

```bash
export KUBECONFIG=/path/to/jaali.yaml

# One-time: register the ArgoCD Application
kubectl apply -f argocd/application-prod.yaml

# Verify
kubectl get application assessment-app-prod -n ci-cd
kubectl get referencegrant allow-routes-assessment-app -n envoy-gateway-system
kubectl get all -n assessment-app-prod
kubectl describe httproute assessment-app -n assessment-app-prod
```

The Application uses **multi-source** sync (`overlays/prod` + `gateway/`) because the ReferenceGrant must be deployed in `envoy-gateway-system`, not `assessment-app-prod`.

## Helm Chart Usage

Alternative to Kustomize — manual install or Harbor OCI distribution:

```bash
export KUBECONFIG=/path/to/jaali.yaml

# Install (matches prod overlay defaults)
helm upgrade --install assessment-app ./chart \
  -n assessment-app-prod \
  --create-namespace \
  --set image.tag="prod-abc12345"

# Dry-run
helm template assessment-app ./chart --set image.tag="prod-abc12345"

# Package and push to Harbor (OCI)
helm package ./chart
helm push assessment-app-0.2.0.tgz oci://harbor.jaali.dev/assessment
```

> ArgoCD deploys via Kustomize by default. The Helm chart is for manual installs, rollbacks, or OCI packaging.

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

## Related Repositories

| Repo | Role |
|------|------|
| [devops-assessment](https://github.com/ThierryMoi/devops-assessment) | Angular app source + Jenkinsfile (CI) |
| [jaali-ai-platform](https://github.com/ThierryMoi/jaali-ai-platform) | Cluster infrastructure (Gateway, ArgoCD, Harbor, Jenkins) |
