# platform-gitops

GitOps infrastructure repository for Kubernetes platform applications. Managed by ArgoCD using the app-of-apps pattern.

## Overview

This repository contains infrastructure application definitions deployed and managed by ArgoCD. Applications are automatically discovered and synced from the `apps/` directory.

### Directory Purpose

**apps/** - ArgoCD Application definitions
- Contains YAML files that define applications to be deployed
- Organized by category:
  - `infrastructure/` - Platform services (Grafana, monitoring, etc.)
  - `domains/` - Business domains (AppProjects and domain app-of-apps)
- Each file creates an ArgoCD Application resource
- ArgoCD automatically discovers new applications through app-of-apps

**manifests/** - Application configuration
- Helm values files, Kustomize overlays, or plain Kubernetes manifests
- Referenced by Application definitions in `apps/`
- Environment-specific configurations
- Custom resource definitions

## Multi-Tenant Organization

This repository demonstrates a multi-tenant GitOps structure with domain-driven organization:

### Infrastructure vs Domain Repositories

**platform-gitops (this repository):**
- Managed by the platform team
- Contains platform infrastructure applications (Grafana, monitoring, operators, etc.)
- Defines ArgoCD AppProjects for RBAC boundaries per domain
- Contains domain app-of-apps that reference external domain repositories

**Domain repositories (payments-gitops, orders-gitops):**
- Organized by business domain (Payments, Orders, Billing, etc.)
- Managed by one or more teams responsible for that domain
- Contains domain-specific applications
- Full autonomy within domain namespace restrictions
- Domains are stable; team ownership can change without restructuring

### ArgoCD Projects (RBAC Boundaries)

**payments-project:**
- Restricts Payments domain to `payments-*` namespaces only
- Can only pull from `git@github.com:AaronStahley/payments-gitops.git`
- Blocks cluster-scoped resources (namespaces created by platform-gitops)
- Prevents accidental deployment to other domains (e.g., Orders)

**orders-project:**
- Restricts Orders domain to `orders-*` namespaces only
- Can only pull from `git@github.com:AaronStahley/orders-gitops.git`
- Blocks cluster-scoped resources (namespaces created by platform-gitops)
- Prevents accidental deployment to other domains (e.g., Payments)

**Benefits:**
- Domain ownership - Domains control their own repositories
- Clear RBAC boundaries - Enforced at ArgoCD and GitHub levels
- Independent CI/CD - Each domain can have separate pipelines
- No cross-domain conflicts - Impossible to modify other domain's apps
- Scalability - Easy to add new domains
- Stability - Domains persist even when teams reorganize

## App-of-Apps Pattern

This repository uses TWO app-of-apps for clear separation between platform and domains:

### Infrastructure App-of-Apps
1. **Root**: The `infra-apps` application is created by platform-bootstrap
2. **Target**: Points to `apps/infrastructure/` in this repository
3. **Discovery**: ArgoCD discovers platform applications (Grafana, Kargo, cert-manager, etc.)
4. **Ownership**: Managed exclusively by platform team

### Domains App-of-Apps
1. **Root**: The `domains-apps` application is created by platform-bootstrap
2. **Target**: Points to `apps/domains/` in this repository with `recurse: true`
3. **Discovery**: ArgoCD discovers domain AppProjects and domain app-of-apps
4. **Structure**: Each domain has subdirectory `apps/domains/{domain-name}/` containing:
   - `project.yaml` - AppProject defining RBAC boundaries
   - `apps.yaml` - Domain app-of-apps pointing to domain's external repository
5. **Domain app-of-apps**: Domain app-of-apps (payments-apps, orders-apps) point to external repositories
6. **Domain applications**: Domain repositories contain their own applications
7. **Automated sync**: Changes at any level are detected and synced automatically

**Benefits of separation:**
- Platform changes don't trigger domain app syncs
- Domain onboarding is isolated from infrastructure changes
- Clear organizational boundaries (by business capability, not team structure)
- Reduced blast radius for changes
- Domains are more stable than team structures

## Adding New Applications

### Infrastructure Applications (Platform Team)

For platform-wide infrastructure services, add to `apps/infrastructure/`:

### Method 1: Helm Chart with Custom Values

Use this approach when deploying a Helm chart with custom configuration.

1. Create an ArgoCD Application definition:

```yaml
# apps/infrastructure/myapp.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "10"
spec:
  project: default
  sources:
    - repoURL: https://charts.example.com
      chart: myapp
      targetRevision: 1.2.3
      helm:
        valueFiles:
          - $values/manifests/myapp/values.yaml
    - repoURL: git@github.com:AaronStahley/platform-gitops.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

2. Create custom Helm values:

```yaml
# manifests/myapp/values.yaml
service:
  type: LoadBalancer
  port: 80

resources:
  limits:
    cpu: 500m
    memory: 512Mi
```

3. Commit and push to the `main` branch
4. ArgoCD will automatically discover and deploy the application

### Method 2: Plain Kubernetes Manifests

For applications that don't use Helm:

```yaml
# apps/infrastructure/myapp.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:AaronStahley/platform-gitops.git
    targetRevision: main
    path: manifests/myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Then place your Kubernetes YAML files in `manifests/myapp/`.

### Domain Applications

For domain-specific applications, domain owners work in domain repositories:

**Payments domain (payments-gitops):**
- Managed by team(s) responsible for Payments domain
- Add applications to `payments-gitops/apps/`
- Add manifests to `payments-gitops/manifests/`
- Applications automatically discovered by `payments-apps` app-of-apps

**Orders domain (orders-gitops):**
- Managed by team(s) responsible for Orders domain
- Add applications to `orders-gitops/apps/`
- Add manifests to `orders-gitops/manifests/`
- Applications automatically discovered by `orders-apps` app-of-apps

### Creating a New Domain

To add a new domain (e.g., "billing"):

1. Create domain directory: `apps/domains/billing/`
2. Create AppProject: `apps/domains/billing/project.yaml`
3. Create domain app-of-apps: `apps/domains/billing/apps.yaml`
4. Commit and push to platform-gitops `main` branch
5. Create external `billing-gitops` repository
6. Add deploy key to ArgoCD

## Application Configuration

### Sync Waves

Control deployment order with sync wave annotations:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "10"
```

Lower numbers deploy first. Use this for dependencies (e.g., CRDs before applications).

### Sync Policies

**Automated sync:**
```yaml
syncPolicy:
  automated:
    prune: true        # Remove resources not in Git
    selfHeal: true     # Revert manual changes
```

**Manual sync:**
```yaml
syncPolicy: {}
```

### Namespace Management

**Platform-managed namespaces:**

Domain namespaces (e.g., `payments-dev`, `payments-qa`) are created by platform-gitops, not by domain applications. This ensures:
- Platform team controls namespace lifecycle
- Consistent namespace configuration across domains
- Proper labels and organization

Namespaces are defined in `manifests/namespaces/` and deployed via the `domain-namespaces` application.

**Infrastructure namespaces:**

Platform infrastructure applications can auto-create their namespaces:
```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true
```

## Example: Grafana Application

The Grafana application demonstrates the multi-source Helm pattern:

**Application Definition:** `apps/infrastructure/grafana.yaml`
- References the official Grafana Helm chart
- Pulls custom values from this repository
- Deploys to `monitoring` namespace

**Custom Values:** `manifests/grafana/values.yaml`
- Configures LoadBalancer service for local access
- Sets resource limits
- Enables persistence
- Configures sample dashboards

## Workflow

### Making Changes

1. Edit application definitions in `apps/` or values in `manifests/`
2. Commit and push to the `main` branch
3. ArgoCD detects changes automatically (default: 3 minutes)
4. Applications sync to match Git state

### Viewing Application Status

```bash
# List all applications
kubectl get applications -n argocd

# View specific application details
kubectl describe application grafana -n argocd

# Check application resources
kubectl get all -n monitoring
```

### Forcing Sync

```bash
kubectl patch application grafana -n argocd --type merge \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}'
```

## GitOps Principles

This repository follows GitOps best practices:

1. **Declarative** - All infrastructure defined in Git as YAML
2. **Versioned** - Full change history via Git commits
3. **Automated** - ArgoCD continuously syncs Git to cluster
4. **Auditable** - Git history provides audit trail
5. **Self-healing** - Manual changes are automatically reverted

## Security Notes

- Never commit sensitive data (passwords, API keys) to this repository
- Use Kubernetes Secrets, Sealed Secrets, or external secret management
- The demo Grafana password (`admin/admin`) is for local development only
- The Kargo admin password hash and token signing key in `manifests/kargo/values.yaml` are also demo values for a local kind cluster; replace them with externally managed secrets for anything beyond a demo
- For production, implement proper secret management solutions

## Related Repositories

**Platform Infrastructure:**
- [platform-bootstrap](https://github.com/AaronStahley/platform-bootstrap) - Cluster setup and ArgoCD
- [platform-gitops](https://github.com/AaronStahley/platform-gitops) - This repository

## Further Reading
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ArgoCD Projects (Multi-tenancy)](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
- [ArgoCD Multi-Source Applications](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)