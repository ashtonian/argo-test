# ArgoCD ApplicationSet Patterns

Three approaches for managing app deployment across a large fleet of clusters using ArgoCD ApplicationSets and Kustomize.

**Goal:** Deploy a base set of apps to all clusters, apply per-group configuration overrides, and selectively deploy additional apps to specific clusters.

---

## Approach 1: Overlay (`overlay/`)

A single ApplicationSet with a **git directory generator** that scans a per-cluster overlay directory. Each subdirectory becomes an ArgoCD Application. Apps are selected by directory presence — if a cluster's overlay doesn't include an app directory, it doesn't get deployed.

### Structure

```
overlay/
├── apps/                              # Base app definitions
│   ├── monitoring/
│   │   ├── deployment.yaml
│   │   └── kustomization.yaml
│   ├── ingress/
│   ├── logging/
│   ├── gpu-operator/                  # Available but not deployed everywhere
│   └── edge-cache/
│
├── clusters/
│   └── homelab/                       # This cluster's app selection
│       ├── monitoring/
│       │   └── kustomization.yaml     # → refs ../../../apps/monitoring
│       ├── ingress/
│       │   └── kustomization.yaml
│       └── logging/
│           └── kustomization.yaml
│       # gpu-operator and edge-cache: not included → not deployed
│
└── applicationset.yaml
```

### How It Works

1. All app definitions live in `apps/` as kustomize bases
2. Each cluster has a directory under `clusters/` containing only the apps it should run
3. Each cluster-app overlay is a `kustomization.yaml` that references the base app (and can add patches)
4. The ApplicationSet scans `overlay/clusters/<cluster>/*` — directory presence = deployment
5. To add an app: create the overlay directory. To remove: delete it (auto-sync prunes it)

### ApplicationSet

```yaml
generators:
  - git:
      repoURL: <repo>
      revision: HEAD
      directories:
        - path: overlay/clusters/homelab/*
```

### Trade-offs

| Pro | Con |
|-----|-----|
| Simple to understand | One ApplicationSet per cluster |
| Easy to add/remove apps per cluster | Doesn't scale well to thousands of clusters |
| Standard kustomize patterns | No built-in cluster parameter injection |
| Overlays can patch base apps | Per-cluster directories needed for every app |

---

## Approach 2: Matrix (`matrix/`)

Two ApplicationSets using the **matrix generator** to combine cluster identity with app discovery. Cluster config files provide parameters (group, server URL, etc.) that feed into the second generator's path templates.

This separates the problem into two layers:
- **Base apps with group overrides** — every cluster gets these, rendered through its group's kustomize overlay
- **Cluster-specific addons** — only clusters that opt in (via directory presence) get these

### Structure

```
matrix/
├── apps/
│   ├── base/                          # Deployed to ALL clusters
│   │   ├── monitoring/
│   │   ├── ingress/
│   │   └── logging/
│   └── addons/                        # Deployed selectively per cluster
│       ├── gpu-operator/
│       └── edge-cache/
│
├── groups/                            # Per-group kustomize overlays
│   ├── us-east/
│   │   ├── config.yaml                # group: us-east
│   │   └── apps/
│   │       ├── monitoring/
│   │       │   └── kustomization.yaml # refs base + group patches
│   │       ├── ingress/
│   │       └── logging/
│   └── eu-west/
│       ├── config.yaml
│       └── apps/
│           ├── monitoring/
│           ├── ingress/
│           └── logging/
│
├── clusters/                          # Per-cluster identity + addon selection
│   ├── cluster-001/
│   │   ├── config.yaml                # cluster_name, group, cluster_server
│   │   └── addons/
│   │       └── gpu-operator/
│   │           └── kustomization.yaml # refs ../../../../apps/addons/gpu-operator
│   ├── cluster-002/
│   │   └── config.yaml                # No addons directory → no extra apps
│   └── cluster-003/
│       ├── config.yaml
│       └── addons/
│           ├── gpu-operator/
│           └── edge-cache/
│
└── applicationsets.yaml               # Two ApplicationSets
```

### Cluster Config

Each cluster declares its identity and which group it belongs to:

```yaml
# clusters/cluster-001/config.yaml
cluster_name: cluster-001
group: us-east
cluster_server: https://cluster-001.example.com
```

The `group` field links the cluster to its group overlay, which controls how base apps are configured for that cluster.

### ApplicationSet 1: Base Apps (all clusters × group overlays)

```yaml
generators:
  - matrix:
      generators:
        # Generator 1: Discover clusters, read their config
        - git:
            files:
              - path: matrix/clusters/*/config.yaml
        # Generator 2: Find base apps for each cluster's group
        # {{.group}} comes from Generator 1
        - git:
            directories:
              - path: 'matrix/groups/{{.group}}/apps/*'
```

**Flow:**
1. Generator 1 reads `clusters/*/config.yaml` → `{cluster_name: cluster-001, group: us-east, ...}`
2. Generator 2 uses `{{.group}}` → scans `groups/us-east/apps/*` → finds `monitoring`, `ingress`, `logging`
3. Matrix merges both parameter sets → generates `cluster-001-monitoring`, `cluster-001-ingress`, etc.
4. Repeat for every cluster — each gets all base apps through its group's overlay

### ApplicationSet 2: Cluster Addons (selective per cluster)

```yaml
generators:
  - matrix:
      generators:
        # Generator 1: Discover clusters, read their config
        - git:
            files:
              - path: matrix/clusters/*/config.yaml
        # Generator 2: Find addons for EACH cluster
        # {{.cluster_name}} comes from Generator 1
        - git:
            directories:
              - path: 'matrix/clusters/{{.cluster_name}}/addons/*'
```

**Flow:**
1. Generator 1 reads cluster configs
2. Generator 2 scans `clusters/<cluster_name>/addons/*` — only finds directories that exist
3. cluster-001 has `addons/gpu-operator/` → gets `cluster-001-gpu-operator`
4. cluster-002 has no `addons/` → gets nothing extra
5. cluster-003 has `addons/gpu-operator/` and `addons/edge-cache/` → gets both

### Generated Applications

| Cluster | Base Apps | Addons | Group |
|---------|-----------|--------|-------|
| cluster-001 | monitoring, ingress, logging | gpu-operator | us-east |
| cluster-002 | monitoring, ingress, logging | _(none)_ | us-east |
| cluster-003 | monitoring, ingress, logging | gpu-operator, edge-cache | eu-west |

**Total: 11 Applications** from 2 ApplicationSets and 3 cluster configs.

### Trade-offs

| Pro | Con |
|-----|-----|
| Single pair of ApplicationSets for all clusters | More indirection in the directory structure |
| Cluster params injected from config files | Requires understanding of matrix generator mechanics |
| Group overlays shared across clusters | Two levels of kustomize (group → base) |
| Scales to thousands of clusters | Per-cluster addon directories still needed for selective apps |
| Adding a cluster = one config.yaml | Group must exist before clusters can reference it |
| Full kustomize power for group overrides | Adding a base app requires a kustomization in every group |

---

## Approach 3: Lovely + Charmap (`lovely/`)

Two ApplicationSets using the **matrix generator** with **argocd-lovely-plugin + charmap** for template variable substitution. Base app manifests contain `<::VAR::>` template markers that get resolved at render time — eliminating the entire per-group overlay directory tree.

This is the key difference from Approach 2: instead of `N_groups × N_base_apps` kustomization overlay files, you have `N_base_apps` base files total. The group-specific values are injected via `source.plugin.env` on each generated Application.

### Structure

```
lovely/
├── apps/
│   ├── base/                          # Deployed to ALL clusters (templated)
│   │   ├── monitoring/
│   │   │   ├── deployment.yaml        # Uses <::GROUP::>, <::CLUSTER_NAME::>
│   │   │   └── kustomization.yaml
│   │   ├── ingress/
│   │   └── logging/
│   └── addons/                        # Deployed selectively per cluster
│       ├── gpu-operator/
│       └── edge-cache/
│
├── clusters/                          # Per-cluster identity + addon selection
│   ├── cluster-001/
│   │   ├── config.yaml                # cluster_name, group, cluster_server
│   │   └── addons/
│   │       └── gpu-operator/
│   │           └── kustomization.yaml
│   ├── cluster-002/
│   │   └── config.yaml
│   └── cluster-003/
│       ├── config.yaml
│       └── addons/
│           ├── gpu-operator/
│           └── edge-cache/
│
└── applicationsets.yaml               # Two ApplicationSets
```

**No `groups/` directory at all.** Group-specific configuration is handled entirely by template variable substitution at render time.

### How It Works

1. Base app manifests use `<::GROUP::>` and `<::CLUSTER_NAME::>` template markers in labels, annotations, config values, etc.
2. The ApplicationSet reads each cluster's `config.yaml` to get `group`, `cluster_name`, and `cluster_server`
3. The matrix generator combines clusters × base app directories
4. Each generated Application passes cluster-specific values to the lovely plugin via `source.plugin.env`
5. At render time, charmap resolves `<::GROUP::>` → `us-east`, `<::CLUSTER_NAME::>` → `cluster-001`, etc.
6. The lovely plugin then runs kustomize on the substituted output

### Templated Manifests

Base app manifests use charmap's `<::VAR::>` syntax:

```yaml
# apps/base/monitoring/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: monitoring
  labels:
    app: monitoring
    group: "<::GROUP::>"
    cluster: "<::CLUSTER_NAME::>"
spec:
  template:
    metadata:
      labels:
        app: monitoring
        group: "<::GROUP::>"
        cluster: "<::CLUSTER_NAME::>"
    spec:
      containers:
        - name: monitoring
          command: ['sh', '-c', 'echo "monitoring on <::CLUSTER_NAME::> (<::GROUP::>)"']
```

### Cluster Config

Same as Approach 2 — one `config.yaml` per cluster:

```yaml
# clusters/cluster-001/config.yaml
cluster_name: cluster-001
group: us-east
cluster_server: https://cluster-001.example.com
```

### ApplicationSet 1: Base Apps (all clusters × base apps, with plugin env)

```yaml
generators:
  - matrix:
      generators:
        - git:
            files:
              - path: lovely/clusters/*/config.yaml
        - git:
            directories:
              - path: lovely/apps/base/*
template:
  spec:
    source:
      path: '{{.path.path}}'
      plugin:
        env:
          - name: GROUP
            value: '{{.group}}'
          - name: CLUSTER_NAME
            value: '{{.cluster_name}}'
```

The `source.plugin.env` block is what makes this work — it passes ApplicationSet template parameters as environment variables to the CMP sidecar. Charmap reads these env vars and substitutes `<::GROUP::>` → `us-east`, etc.

### ApplicationSet 2: Cluster Addons (same as Approach 2)

```yaml
generators:
  - matrix:
      generators:
        - git:
            files:
              - path: lovely/clusters/*/config.yaml
        - git:
            directories:
              - path: 'lovely/clusters/{{.cluster_name}}/addons/*'
```

### Generated Applications

Same result as Approach 2:

| Cluster | Base Apps | Addons | Group |
|---------|-----------|--------|-------|
| cluster-001 | monitoring, ingress, logging | gpu-operator | us-east |
| cluster-002 | monitoring, ingress, logging | _(none)_ | us-east |
| cluster-003 | monitoring, ingress, logging | gpu-operator, edge-cache | eu-west |

**Total: 11 Applications** — but with far fewer files in the repo.

### CMP Configuration Requirement

The lovely plugin CMP must be configured to pass through the additional env vars. Add them to the charmap invocation in the plugin config:

```yaml
# In the CMP plugin.yaml generate command:
charmap --mode flag \
  --set GROUP=${GROUP} \
  --set CLUSTER_NAME=${CLUSTER_NAME} \
```

This is a one-time configuration change to the ArgoCD repo-server, not per-cluster.

### Trade-offs

| Pro | Con |
|-----|-----|
| Fewest files — no per-group overlay directories | Requires argocd-lovely-plugin + charmap CMP |
| Adding a group = just set a value in config.yaml | Template substitution is string-only (no structural transforms) |
| Adding a base app = one directory, all clusters get it | CMP config must declare every passthrough variable |
| Single pair of ApplicationSets for all clusters | Harder to diff rendered output without running charmap |
| Same kustomize features still available | Plugin env must be explicitly wired in ApplicationSet template |
| Template vars compose with kustomize patches | |

---

## Comparison at Scale

For a fleet of **3,000 clusters** across **50 groups** with **20 base apps** and **5 optional addons**:

### File Count

| | Approach 1: Overlay | Approach 2: Matrix | Approach 3: Lovely |
|---|---|---|---|
| Base app definitions | 20 dirs | 20 dirs | 20 dirs |
| Group overlay dirs | N/A | 50 × 20 = **1,000 dirs** | **0** |
| Cluster configs | 3,000 dirs × 20 apps = **60,000 dirs** | 3,000 config files | 3,000 config files |
| Addon dirs | (included above) | ~varies per cluster | ~varies per cluster |
| ApplicationSets | **3,000** (one per cluster) | **2** | **2** |
| **Total directories** | **~60,000** | **~4,000** | **~3,000** |

### Operations

| Operation | Overlay | Matrix | Lovely |
|-----------|---------|--------|--------|
| Add a cluster | Create N app dirs | Create 1 config.yaml | Create 1 config.yaml |
| Add a base app | Touch every cluster dir | Add 1 kustomization per group | Add 1 app dir (all clusters get it) |
| Add a group | N/A | Create group dir + N app kustomizations | Set value in cluster configs |
| Change group config | Edit every cluster's kustomization | Edit group overlay | Edit template or config.yaml |
| Add an addon to a cluster | Create dir in cluster | Create dir in cluster | Create dir in cluster |

### Recommendation

- **Small scale (< 10 clusters):** Approach 1 (Overlay) — simplest, no matrix complexity
- **Medium scale (10–100 clusters):** Approach 2 (Matrix) — scales well, full kustomize power per group
- **Large scale (100+ clusters):** Approach 3 (Lovely) — fewest files, groups are just config values, base apps need no per-group copies
