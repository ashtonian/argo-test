# Matrix Generator Approach

Uses a matrix generator to combine cluster identity (from a config file) with per-cluster app directories. The first generator's parameters feed into the second generator's path template.

## Structure

```
apps/                                    # Base app definitions
  c1-app/
    deployment.yaml
    kustomization.yaml
  c2-app/
    deployment.yaml
    kustomization.yaml
  hello-world/
    deployment.yaml
    kustomization.yaml

clusters/
  homelab/
    config.yaml                          # Cluster identity + params
    apps/
      c2-app/
        kustomization.yaml               # resources: [../../../../apps/c2-app]
      hello-world/
        kustomization.yaml               # resources: [../../../../apps/hello-world]
  production/
    config.yaml
    apps/
      c1-app/
        kustomization.yaml               # resources: [../../../../apps/c1-app]
      c2-app/
        kustomization.yaml               # resources: [../../../../apps/c2-app]
      hello-world/
        kustomization.yaml               # resources: [../../../../apps/hello-world]
```

## Config Files

Each cluster has a `config.yaml` that the git files generator reads. All keys become template parameters.

```yaml
# clusters/homelab/config.yaml
cluster_name: homelab
cluster_server: https://kubernetes.default.svc
target_namespace: default
domain: bitlabyrinth.net
```

```yaml
# clusters/production/config.yaml
cluster_name: production
cluster_server: https://prod-cluster.example.com
target_namespace: production
domain: example.com
```

## ApplicationSet

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: cluster-addons
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          # Generator 1: Discover clusters + read their config
          # Produces: cluster_name, cluster_server, target_namespace, domain
          - git:
              repoURL: https://github.com/ashtonian/argo-test.git
              revision: HEAD
              files:
                - path: "clusters/*/config.yaml"

          # Generator 2: Discover apps for EACH cluster
          # Uses cluster_name from Generator 1 to scope the directory scan
          # Produces: path.path, path.basename, etc.
          - git:
              repoURL: https://github.com/ashtonian/argo-test.git
              revision: HEAD
              directories:
                - path: "clusters/{{.cluster_name}}/apps/*"

  goTemplate: true
  template:
    metadata:
      # Namespaced app name: homelab-c2-app, production-hello-world, etc.
      name: '{{.cluster_name}}-{{.path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/ashtonian/argo-test.git
        targetRevision: HEAD
        path: '{{.path.path}}'
      destination:
        server: '{{.cluster_server}}'
        namespace: '{{.target_namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## How It Works

1. **Generator 1** (git files) scans `clusters/*/config.yaml` and finds two clusters:
   - `{cluster_name: homelab, cluster_server: ..., domain: ...}`
   - `{cluster_name: production, cluster_server: ..., domain: ...}`

2. **Generator 2** (git directories) runs once per Generator 1 result, with `{{.cluster_name}}` substituted:
   - For homelab: scans `clusters/homelab/apps/*` → finds `c2-app`, `hello-world`
   - For production: scans `clusters/production/apps/*` → finds `c1-app`, `c2-app`, `hello-world`

3. **Matrix** merges parameters from both generators for each combination:
   - `{cluster_name: homelab, path.basename: c2-app, ...}`
   - `{cluster_name: homelab, path.basename: hello-world, ...}`
   - `{cluster_name: production, path.basename: c1-app, ...}`
   - `{cluster_name: production, path.basename: c2-app, ...}`
   - `{cluster_name: production, path.basename: hello-world, ...}`

4. **Result**: 5 Applications generated, each with cluster-specific destination and params.

## Comparison

| | Kustomize Overlay (current) | Matrix |
|---|---|---|
| Multi-cluster | Separate ApplicationSet per cluster | Single ApplicationSet for all clusters |
| Cluster params | Hardcoded in template or lovely-plugin | Injected from config.yaml per cluster |
| App selection | Directory presence in cluster overlay | Same — directory presence in cluster/apps/ |
| App naming | `test-c2-app` | `homelab-c2-app`, `production-c2-app` |
| Complexity | Simple | More moving parts, but scales better |

## When to Use Matrix

- Multiple clusters sharing the same app catalog
- Cluster-specific parameters (domain, namespace, server) that vary per environment
- Single ApplicationSet to manage all clusters instead of one per cluster
