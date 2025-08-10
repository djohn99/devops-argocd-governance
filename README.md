# devops-argocd-governance

# DevOps Governance – ArgoCD Deployment Template

This repository contains our **Reusable GitHub Actions Workflow** for deploying services to Kubernetes via **ArgoCD**.  
The workflow dynamically:
- Creates or updates an **ArgoCD Project** (`<service>-<environment>`)
- Creates or updates an **ArgoCD Application**
- Sets ArgoCD **Auto-Sync**, **Prune**, and **Self-Heal**

It is **parameterized**, so any service repository can use it without duplicating logic.

---

## 📦 Repository Structure


---

## 🚀 How It Works

1. **Service repo triggers the template** via `workflow_call`
2. The template workflow:
   - Checks if the **ArgoCD Project** exists → creates if missing
   - Applies the **ArgoCD Application** (idempotent: creates/updates)
   - Points the app to the specified repo, path, and Git revision
   - Enables **automated sync** and **drift correction**
3. ArgoCD detects the application and deploys it immediately

---

## 🔧 Inputs

| Name              | Type   | Required | Description |
|-------------------|--------|----------|-------------|
| `service_name`    | string | ✅       | Name of the service (e.g., `auth-service`) |
| `environment`     | string | ✅       | Target environment (e.g., `dev`, `sit`, `stg`, `uat`) |
| `repo_url`        | string | ✅       | Git repository with manifests/Helm values |
| `target_revision` | string | ✅       | Git branch, tag, or commit SHA |

**Secrets**:
- `KUBE_CONFIG` – base64 encoded kubeconfig with cluster access

---

## 🖼 Example ArgoCD Resources Generated

**Project**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: auth-service-stg
  namespace: argocd
spec:
  description: Project for auth-service-stg
  sourceRepos:
    - '*'
  destinations:
    - namespace: '*'
      server: '*'
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'

## Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: auth-service-stg
  namespace: argocd
spec:
  project: auth-service-stg
  destination:
    namespace: env-stg
    server: https://kubernetes.default.svc
  source:
    repoURL: https://github.com/your-org/valuesFile.git
    path: auth-service/stg
    targetRevision: master
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

### How to Call This Template from a Service Repo

name: Deploy Service via ArgoCD Template

on:
  workflow_dispatch:
    inputs:
      service_name:
        description: "Service name"
        required: true
        type: string
      environment:
        description: "Environment"
        required: true
        type: string
      repo_url:
        description: "Git repo with manifests/values"
        required: true
        type: string
      target_revision:
        description: "Git branch, tag, or commit"
        required: true
        type: string

jobs:
  deploy:
    uses: your-org/devops-governance/.github/workflows/argocd-template.yml@main
    with:
      service_name:    ${{ github.event.inputs.service_name }}
      environment:     ${{ github.event.inputs.environment }}
      repo_url:        ${{ github.event.inputs.repo_url }}
      target_revision: ${{ github.event.inputs.target_revision }}
    secrets:
      KUBE_CONFIG: ${{ secrets.KUBE_CONFIG }}
