# ArgoCD ApplicationSet Test

## Current: Kustomize Overlay Approach

Selective app deployment per cluster using kustomize overlays with an ApplicationSet git directory generator.

### Structure

```
apps/                              # Base app definitions (all apps live here)
  c1-app/                          # Defined but NOT deployed to homelab
    deployment.yaml
    kustomization.yaml
  c2-app/
    deployment.yaml
    kustomization.yaml
  hello-world/
    deployment.yaml
    kustomization.yaml

clusters/
  homelab/                         # Per-cluster overlay — only includes desired apps
    c2-app/
      kustomization.yaml           # resources: [../../../apps/c2-app]
    hello-world/
      kustomization.yaml           # resources: [../../../apps/hello-world]
```

### How It Works

1. `apps/` contains base kustomize definitions for every app
2. `clusters/<cluster>/` contains overlay directories that reference base apps
3. The ApplicationSet scans `clusters/homelab/*` — each subdirectory becomes an ArgoCD Application
4. Apps not referenced by a cluster overlay (like `c1-app`) are simply never deployed

### Adding an App to a Cluster

Create `clusters/homelab/c1-app/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../apps/c1-app
```

### Removing an App from a Cluster

Delete its overlay directory (e.g., `clusters/homelab/c1-app/`). Auto-sync with prune will remove it.

### ApplicationSet

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: test-cluster-addons
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/ashtonian/argo-test.git
        revision: HEAD
        directories:
          - path: clusters/homelab/*
  goTemplate: true
  template:
    metadata:
      name: 'test-{{.path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/ashtonian/argo-test.git
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```
