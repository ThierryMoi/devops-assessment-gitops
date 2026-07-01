# Assessment App — GitOps Repository

Kubernetes deployment manifests and Helm chart for the Assessment App, managed by ArgoCD + Argo Rollouts on the [JaaLi platform](https://github.com/ThierryMoi/jaali-ai-platform).

## Structure

```
devops-assessment-gitops/
├── base/                        ← Shared Kustomize manifests
│   ├── rollout.yaml             ← Argo Rollout (canary base)
│   ├── service.yaml
│   ├── httproute.yaml           ← Gateway API (HTTPRoute, not Ingress)
│   └── kustomization.yaml
│
├── overlays/
│   └── prod/                    ← Production overlay
│       ├── namespace.yaml       ← assessment-app-prod
│       ├── hpa.yaml             ← HPA 3→6 replicas (CPU/RAM)
│       └── kustomization.yaml   ← canary steps, prod-<sha> tag, resources
│
├── gateway/                     ← Cross-namespace Gateway API auth
│   ├── reference-grant.yaml     ← Lives in envoy-gateway-system
│   └── kustomization.yaml
│
├── argocd/
│   ├── application-prod.yaml    ← App sync (multi-source, auto-sync)
│   └── application-argo-rollouts.yaml  ← Rollouts controller install
│
├── chart/                       ← Helm chart v0.4.0 (optional / Harbor OCI)
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── namespace.yaml
│       ├── rollout.yaml
│       ├── hpa.yaml
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
| Argo Rollouts namespace | `argo-rollouts` |
| App namespace | `assessment-app-prod` |
| Hostname | `assessment.jaali.dev` |
| Image registry | `harbor.jaali.dev/assessment/assessment-app` |
| TLS | cert-manager + Let's Encrypt (`assessment.jaali.dev` in `jaali-tls` cert) |

The HTTPRoute in `assessment-app-prod` references `jaali-gateway` in another namespace. A **ReferenceGrant** in `gateway/` authorizes this cross-namespace attachment.

## How It Works

```
Jenkins CI  →  Harbor: assessment-app:prod-<sha>
            →  Git commit: overlays/prod/kustomization.yaml (newTag)
            →  ArgoCD auto-sync (multi-source)
                  ├── overlays/prod  → Rollout, HPA, Service, HTTPRoute, Namespace
                  └── gateway        → ReferenceGrant in envoy-gateway-system
            →  Argo Rollouts canary: 20% → pause → 50% → pause → 100%
```

1. Jenkins builds and pushes `harbor.jaali.dev/assessment/assessment-app:prod-<commit-sha>`
2. Jenkins updates `overlays/prod/kustomization.yaml` via git commit
3. ArgoCD detects the change and syncs the Rollout
4. Argo Rollouts runs a **progressive canary** deployment

## Canary Strategy (Production)

Defined in `overlays/prod/kustomization.yaml`:

| Step | Action |
|------|--------|
| `setWeight: 20` | 20 % of pods on the new image |
| `pause: 2m` | Observation window |
| `setWeight: 50` | 50 % on the new image |
| `pause: 2m` | Observation window |
| `setWeight: 100` | Full rollout |

Safety guards: `maxUnavailable: 0`, `maxSurge: 1`, readiness/liveness probes on `/healthz`.

## HPA (Horizontal Pod Autoscaler)

| Setting | Value |
|---------|-------|
| Target | `Rollout/assessment-app` |
| minReplicas | 3 |
| maxReplicas | 6 |
| CPU target | 70 % |
| Memory target | 80 % |

Requires **metrics-server** on the cluster.

## Image Tagging

Images are **never** tagged `latest` or `stable`:

```
prod-<8-char-commit-sha>   e.g. prod-a1b2c3d4
```

## Prerequisites

- ArgoCD in `ci-cd` namespace
- **Argo Rollouts controller** in `argo-rollouts` namespace
- Envoy Gateway with `jaali-gateway`
- metrics-server (for HPA)
- Harbor project `assessment` with cluster pull access
- DNS `assessment.jaali.dev` → cluster IP
- GitHub PAT registered in ArgoCD (private repo)

```bash
export KUBECONFIG=/path/to/jaali.yaml

# ArgoCD repo credentials
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

# One-time: ArgoCD Applications
kubectl apply -f argocd/application-argo-rollouts.yaml
kubectl apply -f argocd/application-prod.yaml
```

## Deployment Verification

```bash
export KUBECONFIG=/path/to/jaali.yaml

kubectl get application assessment-app-prod argo-rollouts -n ci-cd
kubectl get rollout,hpa,pods -n assessment-app-prod
kubectl argo rollouts get rollout assessment-app -n assessment-app-prod
kubectl get referencegrant allow-routes-assessment-app -n envoy-gateway-system
kubectl describe httproute assessment-app -n assessment-app-prod
```

## Incident Response & Rollback

### During a canary (bad image detected early)

```bash
# Stop canary, keep stable version
kubectl argo rollouts abort assessment-app -n assessment-app-prod

# Or revert to previous ReplicaSet
kubectl argo rollouts undo assessment-app -n assessment-app-prod
```

### After full rollout (100 %)

| Method | Command | Speed |
|--------|---------|-------|
| Argo Rollouts undo | `kubectl argo rollouts undo assessment-app -n assessment-app-prod` | ~30s |
| ArgoCD rollback | `argocd app rollback assessment-app-prod <revision>` | ~1min |
| GitOps revert | `git revert` on `overlays/prod/kustomization.yaml` → push | ~1-2min |

### Automatic protections (no human action)

| Mechanism | Behavior |
|-----------|----------|
| readinessProbe | Unhealthy pods excluded from traffic |
| livenessProbe | Crashed pods restarted by Kubernetes |
| HPA | Scales 3→6 pods under load |
| `maxUnavailable: 0` | Old version stays up during rollout |

> **Note:** No `AnalysisRun` (Prometheus metrics) is configured yet — rollback on silent bugs (200 OK but broken) is **manual**.

## Helm Chart Usage

Alternative to Kustomize — manual install or Harbor OCI distribution:

```bash
helm upgrade --install assessment-app ./chart \
  -n assessment-app-prod \
  --create-namespace \
  --set image.tag="prod-abc12345"

helm package ./chart
helm push assessment-app-0.4.0.tgz oci://harbor.jaali.dev/assessment
```

> ArgoCD deploys via Kustomize by default. Helm is for manual installs or OCI packaging.

## Related Repositories

| Repo | Role |
|------|------|
| [devops-assessment](https://github.com/ThierryMoi/devops-assessment) | Angular app source + Jenkinsfile (CI) |
| [jaali-ai-platform](https://github.com/ThierryMoi/jaali-ai-platform) | Cluster infrastructure (Gateway, ArgoCD, Harbor, Jenkins, TLS) |

## Future Improvements

- AnalysisRun with Prometheus (automatic canary abort on error rate)
- PodDisruptionBudget
- NetworkPolicies
- Image signing (Cosign) + SBOM
- Gateway API traffic splitting (fine-grained canary routing)
