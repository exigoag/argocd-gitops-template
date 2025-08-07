# ArgoCD GitOps Template

A comprehensive GitOps template for managing applications and cluster resources across multiple environments using ArgoCD ApplicationSets.

## Architecture Overview

This repository follows a GitOps pattern where:
- **Applications** are automatically generated based on directory structure
- **Cluster resources** are managed per-cluster using folder-based organization
- **Projects** define resource boundaries and permissions
- **Multi-environment deployments** are supported (dev, stage, prod, etc.)

```
argocd-gitops-template/
├── apps/                           # Application definitions
│   └── {project}/                  # Project namespace
│       └── {app-name}/             # Application name
│           └── {cluster}/          # Target cluster
│               └── {namespace}/    # Target namespace
│                   ├── app.yaml    # Application configuration
│                   └── values.yaml # Helm values override
├── bootstrap/                      # ArgoCD ApplicationSets
│   ├── apps.yaml                   # Main application generator
│   ├── cluster-resources.yaml     # Cluster resource generator
│   └── projects.yaml              # Project generator
├── charts/                         # Helm charts (optional)
├── cluster-resources/              # Cluster-wide resources
│   └── {cluster-name}/             # Per-cluster resources
└── projects/                       # ArgoCD project definitions
    └── {project-name}.yaml
```

## How It Works

### ApplicationSet Magic

The template uses ArgoCD ApplicationSets to automatically discover and create applications based on the directory structure:

1. **Path-based Discovery**: The ApplicationSet scans `apps/**/app.yaml` files
2. **Automatic Parsing**: Extracts project, app name, cluster, and namespace from the path
3. **Dynamic Generation**: Creates ArgoCD Applications with proper configurations

### Directory Structure Pattern

```
apps/{project}/{app-name}/{cluster}/{namespace}/
```

For example:
```
apps/sample-project/sample-app/in-cluster/sample-namespace/
├── app.yaml     # Application configuration
└── values.yaml  # Environment-specific Helm values
```

This creates an application named `sample-app-sample-namespace` that:
- Deploys to cluster `in-cluster`
- Uses namespace `sample-namespace`
- Belongs to project `sample-project`

## Directory Structure Explained

### Apps Directory (`apps/`)

Contains application definitions organized by:
- **Project**: Logical grouping of related applications
- **App Name**: The application identifier
- **Cluster**: Target Kubernetes cluster (must be pre-registered in ArgoCD)
- **Namespace**: Target namespace in the cluster

Each application folder contains:
- `app.yaml`: Application configuration and overrides
- `values.yaml`: Helm values specific to this deployment

### Cluster Resources (`cluster-resources/`)

Stores cluster-wide resources like:
- `ClusterSecretStore`
- `ClusterRole` and `ClusterRoleBinding`

Resources are organized by cluster name:
```
cluster-resources/
├── dev-cluster/
├── stage-cluster/
└── prod-cluster/
```

### Projects (`projects/`)

ArgoCD projects define:
- Resource access permissions
- Source repository whitelist
- Destination cluster/namespace restrictions
- RBAC policies

## Configuration

### Application Configuration (`app.yaml`)

Each application can override default settings:

```yaml
appName: ""              # Override generated app name
destNamespace: ""        # Override target namespace  
destServer: ""           # Override target cluster
repoURL: ""              # Override source repository
srcChart: ""             # Override Helm chart name
srcTargetRevision: ""    # Override Git revision/tag
labels: {}               # Additional labels
annotations:             # Application annotations
  # ArgoCD Image Updater configuration
  argocd-image-updater.argoproj.io/image-list: app=registry.io/app:*
```

### Helm Values Hierarchy

Values are merged in this order (last wins):
1. `apps/{project}/base/values.yaml` (project defaults)
2. `apps/{project}/{app}/base/values.yaml` (app defaults)  
3. `apps/{project}/{app}/{cluster}/{namespace}/values.yaml` (environment-specific)

## Getting Started

### Prerequisites

1. **ArgoCD installed** in your Kubernetes cluster
2. **Clusters registered** in ArgoCD for each environment
3. **Projects created** before deploying applications

### Bootstrap Installation

1. **Fork this repository** and update all references from:
   ```
   https://github.com/exigoag/argocd-gitops-template.git
   ```
   to your repository URL.

2. **Apply the bootstrap Application** to your ArgoCD cluster:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  finalizers:
  - resources-finalizer.argocd.argoproj.io
  labels:
    app.kubernetes.io/name: gitops-bootstrap
  name: gitops-bootstrap
  namespace: argocd
spec:
  destination:
    namespace: argocd
    name: in-cluster
  project: default
  source:
    path: bootstrap
    repoURL: https://github.com/exigoag/argocd-gitops-template.git
    targetRevision: develop
  syncPolicy:
    automated:
      allowEmpty: true
      prune: true
      selfHeal: true
    syncOptions:
    - allowEmpty=true
```

3. **Verify ApplicationSets** are created:
```bash
kubectl get applicationsets -n argocd
```

## Adding New Applications

### Step 1: Create Project (if needed)

```yaml
# projects/my-project.yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-project
  namespace: argocd
spec:
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  destinations:
    - namespace: '*'
      server: '*'
  sourceRepos:
    - '*'
```

### Step 2: Create Application Structure

```bash
mkdir -p apps/my-project/my-app/dev-cluster/my-namespace
```

### Step 3: Configure Application

```yaml
# apps/my-project/my-app/dev-cluster/my-namespace/app.yaml
annotations:
  argocd-image-updater.argoproj.io/image-list: my-app=registry.io/my-app:*
```

```yaml
# apps/my-project/my-app/dev-cluster/my-namespace/values.yaml
image:
  repository: registry.io/my-app
  tag: v1.0.0

resources:
  requests:
    memory: 128Mi
    cpu: 100m
```

## Multi-Environment Deployments

Deploy the same app to different environments:

```
apps/my-project/my-app/
├── dev-cluster/
│   └── my-namespace/
├── stage-cluster/  
│   └── my-namespace/
└── prod-cluster/
    └── my-namespace/
```

Each environment can have different:
- Helm values
- Resource limits
- Image tags
- Configuration