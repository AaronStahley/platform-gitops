# Multi-Repository GitOps Setup Guide

## Overview

This platform-gitops repository contains platform infrastructure and the organizational structure for multi-tenant GitOps. The `apps/domains/` directory contains domain subdirectories, each with:

1. **ArgoCD AppProject** - Defines RBAC boundaries for the domain (project.yaml)
2. **Domain app-of-apps** - Points to external domain-specific repository (apps.yaml)

### Current Structure

```
platform-gitops/ (this repo - platform team manages)
├── apps/
│   ├── infrastructure/
│   │   ├── grafana.yaml              # Platform infrastructure apps
│   │   └── namespaces.yaml           # Domain namespace creation
│   └── domains/
│       ├── payments/
│       │   ├── project.yaml          # Payments RBAC boundaries
│       │   └── apps.yaml             # Points to payments-gitops repo
│       └── orders/
│           ├── project.yaml          # Orders RBAC boundaries
│           └── apps.yaml             # Points to orders-gitops repo
├── manifests/
│   └── namespaces/
│       └── payments.yaml               # payments-dev, payments-qa namespaces
```

### ArgoCD Application Hierarchy

```
platform-bootstrap:
  ├── infra-apps (infrastructure app-of-apps)
  │     ↓ discovers apps/infrastructure/ in platform-gitops
  │     ├── grafana (platform app)
  │     └── domain-namespaces (creates payments-dev, payments-qa, etc.)
  │
  └── domains-apps (domains app-of-apps)
        ↓ discovers apps/domains/ in platform-gitops (recurse: true)
        ├── payments-project (AppProject)
        ├── payments-apps (domain app-of-apps)
        │     ↓ points to git@github.com:AaronStahley/payments-gitops.git
        │     └── (discovers payments apps when repo exists)
        ├── orders-project (AppProject)
        └── orders-apps (domain app-of-apps)
              ↓ points to git@github.com:AaronStahley/orders-gitops.git
              └── (discovers orders apps when repo exists)
```

### AppProject RBAC Boundaries

**payments-project**:
- Can only deploy to `payments-*` namespaces
- Can only pull from `git@github.com:AaronStahley/payments-gitops.git`
- Cannot create cluster-scoped resources (namespaces managed by platform-gitops)
- Payments domain cannot accidentally deploy to other domains

**orders-project**:
- Can only deploy to `orders-*` namespaces
- Can only pull from `git@github.com:AaronStahley/orders-gitops.git`
- Cannot create cluster-scoped resources (namespaces managed by platform-gitops)
- Orders domain cannot accidentally deploy to other domains

**Namespace Management**:
- Domain namespaces are created by platform-gitops via the `domain-namespaces` application
- Platform team controls namespace lifecycle and configuration
- Domain applications deploy into pre-existing namespaces (no CreateNamespace=true)
- Two namespaces per domain: `{domain}-dev` and `{domain}-qa`

## Creating a Domain Repository

### Example: Setting Up payments-gitops

#### Step 1: Create Repository

```bash
# On GitHub: Create new repository payments-gitops
# Clone it locally
git clone git@github.com:AaronStahley/payments-gitops.git
cd payments-gitops
```

#### Step 2: Create Repository Structure

```bash
mkdir -p apps
mkdir -p manifests/payments-api
mkdir -p manifests/payments-web
```

Final structure:
```
payments-gitops/
├── apps/
│   ├── payments-api.yaml
│   └── payments-web.yaml
└── manifests/
    ├── payments-api/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── configmap.yaml
    └── payments-web/
        └── deployment.yaml
```

#### Step 3: Create Application Definitions

Create `apps/payments-api.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "20"
spec:
  project: payments-project
  source:
    repoURL: git@github.com:AaronStahley/payments-gitops.git
    targetRevision: main
    path: manifests/payments-api
  destination:
    server: https://kubernetes.default.svc
    namespace: payments-api
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Create `apps/payments-web.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-web
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "21"
spec:
  project: payments-project
  source:
    repoURL: git@github.com:AaronStahley/payments-gitops.git
    targetRevision: main
    path: manifests/payments-web
  destination:
    server: https://kubernetes.default.svc
    namespace: payments-web
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### Step 4: Create Kubernetes Manifests

Create `manifests/payments-api/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-api
  labels:
    app: payments-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payments-api
  template:
    metadata:
      labels:
        app: payments-api
    spec:
      containers:
      - name: api
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

Create `manifests/payments-api/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-api
  labels:
    app: payments-api
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: http
    protocol: TCP
    name: http
  selector:
    app: payments-api
```

Create `manifests/payments-api/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payments-api-config
data:
  API_ENV: "production"
  LOG_LEVEL: "info"
```

Create `manifests/payments-web/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-web
  labels:
    app: payments-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payments-web
  template:
    metadata:
      labels:
        app: payments-web
    spec:
      containers:
      - name: web
        image: nginx:1.27-alpine
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
```

#### Step 5: Add Deploy Key to ArgoCD

ArgoCD needs SSH access to pull from your domain repository.

```bash
# Generate SSH key for ArgoCD access
ssh-keygen -t ed25519 -f ~/.ssh/argocd_payments -N "" -C "argocd-payments"

# Display public key to add to GitHub
cat ~/.ssh/argocd_payments.pub
```

**Add public key to GitHub:**
1. Go to your payments-gitops repository on GitHub
2. Navigate to Settings → Deploy keys
3. Click "Add deploy key"
4. Title: "ArgoCD Read Access"
5. Paste the public key
6. Check "Allow write access" is **unchecked** (read-only)
7. Click "Add key"

**Add private key to ArgoCD:**

```bash
# Create secret with private key
kubectl create secret generic repo-payments-gitops \
  --from-file=sshPrivateKey=~/.ssh/argocd_payments \
  -n argocd

# Label it as an ArgoCD repository secret
kubectl label secret repo-payments-gitops \
  argocd.argoproj.io/secret-type=repository -n argocd

# Add repository URL to the secret
kubectl patch secret repo-payments-gitops -n argocd \
  --type merge \
  -p '{"stringData":{"type":"git","url":"git@github.com:AaronStahley/payments-gitops.git"}}'
```

#### Step 6: Commit and Push

```bash
# Add all files
git add .

# Commit
git commit -m "Initial payments-gitops setup with payments-api and payments-web"

# Push to GitHub
git push origin main
```

#### Step 7: Verify in ArgoCD

```bash
# Check that payments-apps is now healthy
kubectl get application payments-apps -n argocd

# List all payments applications
kubectl get applications -n argocd | grep payments

# Check sync status
kubectl describe application payments-apps -n argocd
```

Expected applications:
- `payments-apps` - Domain app-of-apps (should show Synced/Healthy)
- `payments-api` - Payments API application (should auto-deploy)
- `payments-web` - Payments web application (should auto-deploy)

Verify namespaces created:
```bash
kubectl get namespaces | grep payments
# Expected: payments-api, payments-web
```

Verify pods running:
```bash
kubectl get pods -n payments-api
kubectl get pods -n payments-web
```

## Creating Orders Domain (orders-gitops)

Follow the same steps as Payments, replacing:
- Repository name: `payments-gitops` → `orders-gitops`
- Project reference: `payments-project` → `orders-project`
- Application names: `payments-*` → `orders-*`
- Namespaces: `payments-*` → `orders-*`

Example applications for Orders domain:
- `orders-processor` - Backend processing service
- `orders-portal` - User-facing orders portal

SSH key for Orders:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/argocd_orders -N "" -C "argocd-orders"

# Add public key to GitHub orders-gitops repo as deploy key
# Add private key to ArgoCD
kubectl create secret generic repo-orders-gitops \
  --from-file=sshPrivateKey=~/.ssh/argocd_orders \
  -n argocd

kubectl label secret repo-orders-gitops \
  argocd.argoproj.io/secret-type=repository -n argocd

kubectl patch secret repo-orders-gitops -n argocd \
  --type merge \
  -p '{"stringData":{"type":"git","url":"git@github.com:AaronStahley/orders-gitops.git"}}'
```

## Production Multi-Repo Structure

After both domains are set up:

```
platform-gitops/         (Platform team manages)
├── apps/
│   ├── infrastructure/
│   │   └── grafana.yaml       # Platform services
│   └── domains/
│       ├── payments/
│       │   ├── project.yaml
│       │   └── apps.yaml
│       └── orders/
│           ├── project.yaml
│           └── apps.yaml

payments-gitops/               (Payments domain owners manage)
├── apps/
│   ├── payments-api.yaml
│   └── payments-web.yaml
└── manifests/
    ├── payments-api/
    │   ├── deployment.yaml
    │   ├── service.yaml
    │   └── configmap.yaml
    └── payments-web/
        └── deployment.yaml

orders-gitops/                (Orders domain owners manage)
├── apps/
│   ├── orders-processor.yaml
│   └── orders-portal.yaml
└── manifests/
    ├── orders-processor/
    │   └── deployment.yaml
    └── orders-portal/
        └── deployment.yaml
```

## Benefits of Multi-Repo GitOps

### Domain Ownership
- Each domain has its own repository
- Domain owners control their repository (may be one or more teams)
- Domains cannot accidentally modify other domain's applications
- Clear separation of responsibilities by business capability

### Independent RBAC
- GitHub repository permissions control who can modify each domain
- Payments domain owners have write access to payments-gitops only
- Orders domain owners have write access to orders-gitops only
- Platform team manages platform-gitops
- Team reorganizations don't require repository restructuring

### Separate CI/CD Pipelines
- Each domain can implement their own CI/CD workflows
- Payments can use different testing/deployment strategies than Orders
- Domains iterate at their own pace

### ArgoCD Project Boundaries
- `payments-project` restricts Payments apps to `payments-*` namespaces
- `orders-project` restricts Orders apps to `orders-*` namespaces
- Impossible to deploy to wrong namespace due to AppProject restrictions

### Scalability
- Easy to add new domains (add AppProject + domain app-of-apps + create repo)
- No cross-domain coordination needed for application changes
- Each domain repository stays focused and manageable

### Stability
- Domains based on business capabilities, not team structure
- Team reorganizations don't require repository changes
- A team can own multiple domains, or multiple teams can share a domain
- Independent versioning - domains work at different speeds

## Domain Workflow Example

### Payments Domain Contributor Workflow

```bash
# Clone payments domain repository
cd ~/projects
git clone git@github.com:AaronStahley/payments-gitops.git
cd payments-gitops

# Create new application
cp apps/payments-api.yaml apps/payments-billing.yaml

# Edit for new service
vim apps/payments-billing.yaml
# Change: name, path, namespace

# Create manifests
mkdir -p manifests/payments-billing
vim manifests/payments-billing/deployment.yaml

# Commit and push
git add .
git commit -m "Add payments-billing application"
git push origin main

# ArgoCD automatically detects and deploys
# Check status
kubectl get applications -n argocd | grep payments-billing
```

### Platform Team Workflow

```bash
# Clone infrastructure repository
cd ~/projects
git clone git@github.com:AaronStahley/platform-gitops.git
cd platform-gitops

# Add new domain (e.g., Billing)
# 1. Create domain directory
mkdir -p apps/domains/billing

# 2. Create AppProject
vim apps/domains/billing/project.yaml

# 3. Create domain app-of-apps
vim apps/domains/billing/apps.yaml

# 4. Commit and push
git add apps/domains/billing/
git commit -m "Add Billing domain"
git push origin main

# 5. Domain owners create billing-gitops repository separately
# 6. Platform team adds deploy key to ArgoCD
```

## Adding a New Domain

To add a new domain (e.g., "billing"):

### 1. Platform Team: Create AppProject and App-of-Apps

In `platform-gitops`:

Create directory structure:
```bash
mkdir -p apps/domains/billing
```

Create `apps/domains/billing/project.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: billing-project
  namespace: argocd
spec:
  description: Billing domain applications and resources
  sourceRepos:
  - 'git@github.com:AaronStahley/billing-gitops.git'
  destinations:
  - namespace: 'billing-*'
    server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist: []
  namespaceResourceWhitelist:
  - group: ''
    kind: '*'
  - group: 'apps'
    kind: '*'
  - group: 'batch'
    kind: '*'
  - group: 'networking.k8s.io'
    kind: '*'
```

Create `apps/domains/billing/apps.yaml`:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: billing-apps
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "5"
spec:
  project: billing-project
  source:
    repoURL: git@github.com:AaronStahley/billing-gitops.git
    targetRevision: main
    path: apps
    directory:
      recurse: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Commit and push:
```bash
git add apps/domains/billing/
git commit -m "Add billing domain"
git push origin main
```

### 2. Domain Owners: Create Domain Repository

Follow the steps in "Creating a Domain Repository" section above, replacing "payments" with "billing".

### 3. Platform Team: Add Deploy Key

Follow the SSH key setup in Step 5, creating `argocd_billing` key and adding to ArgoCD.

## Troubleshooting

### Domain app-of-apps shows "ComparisonError: repository not found"

**Expected behavior** until domain repository exists:
- `payments-apps` will show error if `payments-gitops` repo doesn't exist yet
- Create the domain repository and add deploy key to resolve

### Application fails with "permission denied"

**Check AppProject restrictions:**
```bash
kubectl get appproject payments-project -n argocd -o yaml
```

Common issues:
- Trying to deploy to namespace not matching project's destination pattern
- Repository URL not in project's sourceRepos list
- Trying to create cluster-scoped resources (not in whitelist)

### Deploy key issues

**Verify secret exists:**
```bash
kubectl get secret repo-payments-gitops -n argocd
kubectl get secret repo-payments-gitops -n argocd -o yaml
```

**Check ArgoCD repository status:**
```bash
# ArgoCD UI: Settings → Repositories
# Or via CLI:
kubectl get secrets -n argocd -l argocd.argoproj.io/secret-type=repository
```

### Application not auto-syncing

**Check sync policy:**
```bash
kubectl get application payments-apps -n argocd -o yaml | grep -A5 syncPolicy
```

Should have:
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

## Reference

- [ArgoCD Projects Documentation](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
- [ArgoCD App-of-Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [ArgoCD Private Repositories](https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/)
