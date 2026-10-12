# Design decisions

Short records of the main choices behind the platform: the context, what was decided, why, and what it costs. Newer decisions go at the end.

## 1. A custom operator instead of a shared Helm chart

**Context.** Teams need a simple way to run a web service. The usual starting point is a shared Helm chart that every team fills in with values.

**Decision.** Developers declare a `WebApi` custom resource (`platform.aaronstahley.dev/v1alpha1`), and a Go operator reconciles it into a Deployment and Service.

**Why.**
- **A real API with validation.** The schema is typed, and CEL rules reject bad input when it is applied, not after a failed rollout: the `latest` tag is refused, and probes require a container port.
- **Status developers can read.** The operator reports `Ready`, `Progressing` and `Degraded` on the resource itself, so "is my change live?" has one place to look.
- **Room to change the implementation.** The platform team can change what the operator generates behind the same `WebApi` without every team editing values files.
- **Continuous reconciliation.** Drift in the generated objects is corrected, not just applied once.

**Cost.** Go code, tests and releases to maintain, and CRD versioning to manage once the API changes. A Helm chart is quicker to start with and easier to read for small setups.

## 2. GitOps with Argo CD, and promotion with Kargo

**Context.** Changes need an audit trail, a controlled path from dev to prod, and no direct `kubectl` access to environments.

**Decision.** Git is the only way to change an environment. Argo CD syncs each environment; Kargo promotes changes between them by rendering manifests into `stage/<env>` branches.

**Why.**
- **Git is the audit log.** Every change to every environment is a commit with an author and a diff.
- **Promotion is an explicit object.** Kargo packages a manifest commit and an image tag as Freight. qa only accepts Freight that has passed dev, and prod only Freight from qa, so the order cannot be skipped by accident.
- **Environments are diffable.** Because each stage is a rendered branch, "what is different between qa and prod" is a `git diff`.
- **dev is automatic, qa and prod are deliberate.** dev auto-promotes new Freight; later stages are promoted by a person.

**Cost.** More moving parts than a CI job that runs `kubectl apply`: two controllers, stage branches and their credentials. Promotions need a token that can push to the domain repos.

## 3. One repository per domain, with AppProjects as tenant boundaries

**Context.** Several teams share the cluster and must not affect each other.

**Decision.** Each domain (`payments`, `orders`) has its own `<domain>-gitops` repo and its own Argo CD AppProject. The platform team owns this repo, the namespaces and the AppProjects.

**Why.**
- **Ownership matches access.** A team's repo permissions decide who can change its services.
- **Enforced boundaries.** Each AppProject only allows its own repo and its own `<domain>-*` namespaces, so one domain cannot deploy into another.
- **Domains, not teams.** Domains outlive team reorganisations, so repos do not have to move when teams change.

**Cost.** More repos and credentials to manage, and cross-domain changes take several pull requests. The sample tenants run podinfo in place of real applications.

## 4. A local kind cluster as the reference environment

**Context.** The platform has to be easy to try and to reset.

**Decision.** `platform-bootstrap` builds everything on a local kind cluster with `make all`, then hands over to Argo CD.

**Why.** Reproducible in about ten minutes on a laptop, free, and easy to throw away. Bootstrap does only day-0 work; everything after that is managed by this repo.

**Cost.** No real cloud networking, load balancers or identity. Current kind node images also require a cgroup v2 host: on WSL2 with Docker Desktop that means one `.wslconfig` setting, documented in the bootstrap README.

## 5. Bootstrap waits on every application, not the root

**Context.** The first bootstrap reported success while nothing had deployed. A root app-of-apps reports `Healthy` even when its children fail, and an application that cannot load its source also reports `Healthy` with sync status `Unknown`.

**Decision.** Bootstrap waits until every Argo CD application is `Synced` and `Healthy`, prints each application's error if it times out, and only skips applications whose `stage/<env>` branch does not exist yet.

**Why.** Success should mean the platform is running. Without this check, credential and permission errors stay hidden until someone looks at the UI.

**Cost.** A slightly longer wait during bootstrap, and some shell logic to maintain.
