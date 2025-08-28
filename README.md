# ArgoCD GitOps Template

A GitOps template for managing apps and cluster resources with ArgoCD ApplicationSets. It supports a branch-driven workflow (develop vs main) that works well with Renovate and staged promotions.

## Highlights
- Applications are auto-generated from folders and a branch-prefixed app manifest
- Separate generators per Git branch (develop, main)
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
│                   ├── <branch>.app.yaml  # One per leaf; e.g., develop.app.yaml OR main.app.yaml
│                   └── values.yaml        # Helm values for this env
├── bootstrap/                      # ArgoCD ApplicationSets
│   ├── apps.yaml                   # Main application generator (per-branch)
│   ├── cluster-resources.yaml      # Cluster-wide resources generator
│   └── projects.yaml               # Project generator (optional)
├── charts/                         # Optional local Helm charts
├── cluster-resources/              # Cluster-scoped resources
│   └── {cluster-name}/             # One folder per cluster
└── projects/                       # ArgoCD AppProject specs
    └── {project-name}.yaml
```

Note: Typically you have exactly one `<branch>.app.yaml` per environment folder. Use `develop.app.yaml` for non-prod (dev/stage) and `main.app.yaml` for prod, each in their respective cluster/namespace folders.

## How Application Generation Works

If you're new to ApplicationSets, think of it like this: one Argo CD Application is created for each leaf folder under `apps/{project}/{app}/{cluster}/{namespace}`. The folder names tell Argo CD where to deploy, and a small YAML file in that folder provides optional overrides.

Directory-to-fields mapping:
- project = `apps/{project}/...`
- app name = `{app-name}` (also default Helm chart name)
- cluster = `{cluster}` (must match a cluster pre-added to Argo CD)
- namespace = `{namespace}` (target namespace in the cluster)

Example
```
apps/sample-project/sample-app/in-cluster/sample-namespace/
├── develop.app.yaml
└── values.yaml
```
Generates an Argo CD Application that:
- belongs to project `sample-project`
- targets cluster `in-cluster`
- deploys to namespace `sample-namespace`
- is named `sample-app-sample-namespace` (unless overridden in the app file)

### Note on branches (optional)
If you keep multiple branches, name your app file accordingly, e.g.:
- `develop.app.yaml` → tracks the `develop` branch
- `main.app.yaml` → tracks the `main` branch

To support another branch, add a generator in `bootstrap/apps.yaml` that matches `apps/**/<branch>.app.yaml` and sets `values.revision` to that branch.

### Source Selection and Overrides
By default the Application sources from this repo and derives sensible defaults from the path, but you can override in the app file:
- `repoURL`: source Git repo or Helm registry
- `targetRevision`: comes from the generator (e.g., develop/main) or `srcTargetRevision`
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
          values.yaml     # 3) env-specific overrides
          develop.app.yaml
```

## App File Schema (develop.app.yaml / main.app.yaml)
Place in: `apps/{project}/{app}/{cluster}/{namespace}/`

```yaml
appName: ""           # Override app name (default: {app-name}-{namespace-directory})
destNamespace: ""     # Override target namespace (default: {namespace-directory})
destServer: ""        # Override target cluster (default: {cluster-directory})
repoURL: ""           # Override source repo (default: this repo)
srcChart: ""          # Override Helm chart (default: {app-name})
srcTargetRevision: "" # Override Git branch/tag (default: from the generator, e.g., develop or main)
labels: {}             # Extra labels
annotations:           # Annotations (e.g., Argo CD Image Updater)
  # argocd-image-updater.argoproj.io/image-list: app=registry.example.com/app:*
```

Tip: You can use both `develop.app.yaml` and `main.app.yaml` in the repository, but place them in the appropriate environment folders (e.g., `develop.app.yaml` under dev/stage clusters and `main.app.yaml` under prod clusters). Typically, there is one app file per leaf folder.

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

Example structure for a staging environment:
```
apps/my-project/my-app/dev-cluster/my-namespace/
├── develop.app.yaml
└── values.yaml
```

develop.app.yaml
```yaml
# apps/my-project/my-app/dev-cluster/my-namespace/develop.app.yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: my-app=registry.example.com/my-app:*
```

values.yaml
```yaml
# apps/my-project/my-app/dev-cluster/my-namespace/values.yaml
image:
  repository: registry.example.com/my-app
  tag: v1.2.3
```

When ready for production, create a prod folder:
```
apps/my-project/my-app/prod-cluster/my-namespace/
└── main.app.yaml
```

main.app.yaml
```yaml
# apps/my-project/my-app/prod-cluster/my-namespace/main.app.yaml
repoURL: oci://ghcr.io/YOUR-ORG/charts
srcChart: my-app
# srcTargetRevision: 1.2.3
```

## Add Support for Another Branch
Edit `bootstrap/apps.yaml` and add another generator matching your filename, e.g. `release.app.yaml`, and set:
- `files: - path: "apps/**/release.app.yaml"`
- `values.revision: "release"`

Commit and sync. Any `release.app.yaml` files will now render Applications from the `release` branch.

## Best Practices
- Keep app and namespace names short and kebab-case
- Use Renovate to update charts/images in `develop` first, then promote by merging to `main`
- Keep per-env differences in `values.yaml`

## Troubleshooting
- App didn’t appear: check the folder depth and that the app file exists
- Wrong branch: confirm the filename matches `<branch>.app.yaml` or set `srcTargetRevision`
- Cluster mismatch: ensure the Argo CD cluster name matches your `{cluster}` folder
- Project not found: make sure the `AppProject` exists in Argo CD before apps render

## Notes
- Update all repo URLs to your own (bootstrap and any references in ApplicationSets)
- Clusters must be pre-added to Argo CD to use their names in directory paths
- Projects can be managed here but must exist prior to Application creation

## License
See [LICENSE](LICENSE).