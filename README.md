# ArgoCD GitOps Template

A GitOps template for managing apps and cluster resources with ArgoCD ApplicationSets. It supports cluster-based deployments with Renovate handling staged promotions between nonprod and production environments.

## Highlights
- Applications are auto-generated from folders with a single app manifest per deployment
- Renovate manages staged deployments based on cluster names (nonprod vs prod)
- Cluster-wide resources managed per cluster
- ArgoCD Projects isolate and govern apps

## Repository Layout
```
argocd-gitops-template/
├── apps/                           # Application definitions
│   └── {project}/                  # Project name
│       └── {app-name}/             # Application name
│           └── {cluster}/          # Target cluster (must exist in ArgoCD)
│               └── {namespace}/    # Target namespace
│                   ├── app.yaml            # Application configuration
│                   └── values.yaml         # Helm values for this env
├── bootstrap/                      # ArgoCD ApplicationSets
│   ├── apps.yaml                   # Main application generator
│   ├── cluster-resources.yaml      # Cluster-wide resources generator
│   └── projects.yaml               # Project generator (optional)
├── charts/                         # Optional local Helm charts
├── cluster-resources/              # Cluster-scoped resources
│   └── {cluster-name}/             # One folder per cluster
└── projects/                       # ArgoCD AppProject specs
    └── {project-name}.yaml
```

Note: Each deployment folder contains a single `app.yaml` file. Renovate handles deployment staging by treating clusters differently based on their names (nonprod vs prod clusters).

## How Application Generation Works

If you're new to ApplicationSets, think of it like this: one Argo CD Application is created for each leaf folder under `apps/{project}/{app}/{cluster}/{namespace}`. The folder names tell Argo CD where to deploy, and the `app.yaml` file in that folder provides configuration and overrides.

Directory-to-fields mapping:
- project = `apps/{project}/...`
- app name = `{app-name}` (also default Helm chart name)
- cluster = `{cluster}` (must match a cluster pre-added to Argo CD)
- namespace = `{namespace}` (target namespace in the cluster)

Example
```
apps/sample-project/sample-app/in-cluster/sample-namespace/
├── app.yaml
└── values.yaml
```
Generates an Argo CD Application that:
- belongs to project `sample-project`
- targets cluster `in-cluster`
- deploys to namespace `sample-namespace`
- is named `sample-project-sample-app-in-cluster-sample-namespace` (deduplicated and lowercased, unless overridden)

### Renovate-Based Staging
Renovate handles deployment staging by treating clusters differently based on their names:
- **Nonprod clusters**: Matched by `/dev-cluster/` pattern - auto-merge enabled, 2 days minimum release age
- **Prod clusters**: Matched by `/in-cluster/` and `/prod-cluster/` patterns - manual approval required, 7 days minimum release age

This configuration is defined in `renovate.json` and eliminates the need for branch-based workflows.

### Source Selection and Overrides
By default the Application sources from this repo (main branch) and derives sensible defaults from the path, but you can override in the app file:
- `repoURL`: source Git repo or Helm registry
- `srcTargetRevision`: Git branch/tag (default: main)
- If `repoURL` ends with `.git`, `srcChart` is treated as a path; otherwise as a registry chart name

Helm value files are merged (later wins):
1) `apps/{project}/base/values.yaml`
2) `apps/{project}/{app}/base/values.yaml`
3) `apps/{project}/{app}/{cluster}/{namespace}/values.yaml`

Values overlay in the generator:
```yaml
valueFiles:
  - "$repo/apps/{{ .values.project }}/base/values.yaml"
  - "$repo/{{ .values.base_path }}/values.yaml"
  - "$repo/{{ .values.app_path }}/values.yaml"
```

Visual example (last wins):
```
apps/
  my-project/
    base/
      values.yaml         # 1) organization/project defaults
    my-app/
      base/
        values.yaml       # 2) app defaults
      dev-cluster/
        my-namespace/
          app.yaml        # 3) deployment-specific config
          values.yaml     # 3) deployment-specific overrides
```

## App File Schema (app.yaml)
Place in: `apps/{project}/{app}/{cluster}/{namespace}/`

```yaml
appName: ""           # Override app name (default: {project}-{app-name}-{cluster}-{namespace} deduplicated and lowercased)
destNamespace: ""     # Override target namespace (default: {namespace-directory})
destServer: ""        # Override target cluster (default: {cluster-directory})
repoURL: ""           # Override source repo (default: this repo)
srcChart: ""          # Override Helm chart (default: {app-name})
srcTargetRevision: "" # Override Git branch/tag (default: main)
labels: {}            # Additional Application labels
annotations:          # Annotations (e.g., Argo CD Image Updater)
  # argocd-image-updater.argoproj.io/image-list: app=registry.example.com/app:*
```

Each deployment folder contains exactly one `app.yaml` file that defines the application configuration for that specific cluster/namespace combination.

## Cluster Resources
Put cluster-scoped resources under `cluster-resources/{cluster-name}/`, for example:
- `ClusterSecretStore`, cluster RBAC, global policies, CRDs, webhooks

These are synced per cluster by `bootstrap/cluster-resources.yaml`.

## Projects
ArgoCD `AppProject` specs live in `projects/`. They:
- Restrict destinations and sources
- Whitelist resource kinds
- Apply RBAC

Projects can be defined here, but must exist in ArgoCD before Applications referencing them are generated.

## Prerequisites
- ArgoCD installed in your cluster
- Clusters pre-registered in ArgoCD (names must match your `{cluster}` folder names)
- Projects created or managed here and applied before apps render

### Adding Clusters
When adding clusters to ArgoCD, use short descriptive names to keep application names readable:

```bash
# Good: Short, descriptive names
argocd cluster add my-k8s-context --name dev-cluster
argocd cluster add my-prod-context --name prod-cluster

# Avoid: Long server URLs as names (default behavior)
# This would create very long application names
```

The cluster name you specify with `--name` becomes part of the generated application name, so choose wisely!

## Bootstrap
Apply this Application to the ArgoCD cluster and update the repo URL to yours:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  finalizers:
  - resources-finalizer.argocd.argoproj.io
  labels:
    app.kubernetes.io/name: exigitops
  name: exigitops
  namespace: argocd
spec:
  destination:
    namespace: argocd
    name: in-cluster
  project: default
  source:
    path: bootstrap
    repoURL: https://github.com/YOUR-ORG/argocd-gitops-template.git
  syncPolicy:
    automated:
      allowEmpty: true
      prune: true
      selfHeal: true
    syncOptions:
    - allowEmpty=true
```

After syncing, verify ApplicationSets:
```bash
kubectl get applicationsets -n argocd
```

## Add a New App (Example)

Example structure for a nonprod deployment using podinfo:
```
apps/sample-project/podinfo/dev-cluster/podinfo-ns/
├── app.yaml
└── values.yaml
```

app.yaml
```yaml
# apps/sample-project/podinfo/dev-cluster/podinfo-ns/app.yaml
repoURL: "https://stefanprodan.github.io/podinfo"
srcChart: "podinfo"
srcTargetRevision: "6.9.1"
```

values.yaml
```yaml
# apps/sample-project/podinfo/dev-cluster/podinfo-ns/values.yaml
replicaCount: 1
resources:
  requests:
    cpu: 10m
    memory: 16Mi
```

For production, create a prod cluster deployment:
```
apps/sample-project/podinfo/prod-cluster/podinfo-ns/
├── app.yaml
└── values.yaml
```

app.yaml
```yaml
# apps/sample-project/podinfo/prod-cluster/podinfo-ns/app.yaml
repoURL: "https://stefanprodan.github.io/podinfo"
srcChart: "podinfo"
srcTargetRevision: "6.9.1"
```

values.yaml
```yaml
# apps/sample-project/podinfo/prod-cluster/podinfo-ns/values.yaml
replicaCount: 3
resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

Renovate will automatically:
- Deploy updates to nonprod clusters (`dev-cluster`) with auto-merge after 2 days
- Create PRs for prod clusters (`in-cluster`, `prod-cluster`) requiring manual approval after 7 days

## Renovate Configuration

Renovate handles deployment staging automatically based on cluster naming patterns defined in `renovate.json`:

```json
{
  "packageRules": [
    {
      "matchFileNames": ["**/dev-cluster/**"],
      "labels": ["env:nonprod"],
      "groupName": "nonprod",
      "automerge": true,
      "minimumReleaseAge": "2 days",
      "prPriority": 20
    },
    {
      "matchFileNames": ["**/in-cluster/**", "**/prod-cluster/**"],
      "labels": ["env:prod"],
      "groupName": "prod", 
      "automerge": false,
      "minimumReleaseAge": "7 days",
      "prPriority": 0
    }
  ]
}
```

**Nonprod Clusters** (glob pattern: `**/dev-cluster/**`):
- Labeled with `env:nonprod`
- Auto-merge enabled
- 2 days minimum release age
- Higher priority (20)

**Prod Clusters** (glob patterns: `**/in-cluster/**`, `**/prod-cluster/**`):
- Labeled with `env:prod` 
- Manual approval required (auto-merge disabled)
- 7 days minimum release age
- Lower priority (0)

To add new cluster types, update the `packageRules` with appropriate `matchFileNames` patterns.

## Best Practices
- Keep app and namespace names short and kebab-case
- Use Renovate's automatic staging: updates flow from nonprod to prod clusters based on cluster naming
- Keep deployment-specific differences in `values.yaml`
- Name clusters according to the Renovate patterns for proper staging behavior

## Troubleshooting
- App didn't appear: check the folder depth and that the `app.yaml` file exists
- Cluster mismatch: ensure the Argo CD cluster name matches your `{cluster}` folder
- Project not found: make sure the `AppProject` exists in Argo CD before apps render

## Notes
- Update all repo URLs to your own (bootstrap and any references in ApplicationSets)
- Clusters must be pre-added to Argo CD to use their names in directory paths
- Projects can be managed here but must exist prior to Application creation

## License
See [LICENSE](LICENSE).