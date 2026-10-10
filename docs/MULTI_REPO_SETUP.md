# Multi-Repository GitOps Setup

How the platform repo and the domain repos fit together, and how to add a domain.

## Repositories

| Repo | Owner | Contains |
|---|---|---|
| `platform-gitops` (this repo) | Platform team | Platform apps (Kargo, cert-manager, Grafana, the operator per env), domain namespaces, and one AppProject + app-of-apps per domain |
| `<domain>-gitops` | Domain team | The domain's `WebApi` manifests, its Argo CD Applications per environment, and its Kargo project, warehouse and stages |
| `platform-bootstrap` | Platform team | Day-0 only: kind cluster, Argo CD, credentials, the two root app-of-apps |

## Argo CD hierarchy

```
platform-bootstrap
├── infra-apps            -> platform-gitops apps/infrastructure/
│   ├── cert-manager, kargo, kargo-rbac, grafana
│   ├── domain-namespaces  (payments-{dev,qa,prod}, orders-{dev,qa,prod})
│   └── webapi-operator-{dev,qa,prod} + webapi-operator-kargo
└── domains-apps          -> platform-gitops apps/domains/ (recurse)
    ├── payments-project   (AppProject)
    ├── payments-apps      -> payments-gitops apps/ (recurse)
    │   ├── payments-kargo        -> kargo/            (Kargo project, warehouse, stages)
    │   └── payments-api-{env}    -> stage/<env> branch, manifests/payments-api/overlays/<env>
    ├── orders-project
    └── orders-apps        -> orders-gitops apps/      (same shape)
```

All repos are referenced over HTTPS. Argo CD reads them with the token that `platform-bootstrap` registers; see its README for the required permissions.

## Domain repo layout

```
payments-gitops/
├── apps/
│   ├── kargo.yaml                  # Argo CD app for the Kargo resources
│   └── {dev,qa,prod}/payments-api.yaml
├── kargo/
│   ├── project.yaml, projectconfig.yaml
│   ├── warehouses/payments-api.yaml
│   └── stages/{dev,qa,prod}.yaml
└── manifests/payments-api/
    ├── base/webapi.yaml            # the WebApi (podinfo)
    └── overlays/{dev,qa,prod}/     # replicas, resources, UI message per env
```

## Promotion flow

1. A change merges to `main` in the domain repo (manifests under `manifests/payments-api/`), or a new podinfo image tag is published.
2. The Kargo **Warehouse** turns the Git commit plus the image tag into **Freight**.
3. Promoting Freight into a **Stage** clones the source commit, sets `spec.image.tag` on the `WebApi`, renders base and the stage's overlay into the `stage/<env>` branch and pushes it.
4. The `payments-api-<env>` Argo CD Application tracks `stage/<env>`, syncs, and the operator reconciles the `WebApi`.

dev is fed straight from the Warehouse; qa only accepts Freight that has been through dev, and prod only Freight from qa. The `stage/qa` and `stage/prod` branches do not exist until the first promotion into those stages, so their Argo CD apps show sync status `Unknown` until then.

## AppProject boundaries

Each domain's AppProject (`apps/domains/<domain>/project.yaml`):

- **Sources:** only the domain's own repo.
- **Destinations:** `<domain>-*` namespaces, the `<domain>` Kargo project namespace, and `argocd` (for the domain's own Application objects).
- **Cluster-scoped resources:** only the domain's Kargo `Project`. Namespaces are created by the platform (`domain-namespaces`), not by domains.
- **Namespaced resources:** core, `apps`, `batch`, `networking.k8s.io`, `autoscaling`, Argo CD `Application`, Kargo resources and `WebApi`.

A domain therefore cannot deploy into another domain's namespaces or pull from another domain's repo.

## Adding a domain

Using `billing` as the example.

**Platform team, in `platform-gitops`:**

1. Copy `apps/domains/payments/` to `apps/domains/billing/` and replace `payments` with `billing` in `project.yaml` and `apps.yaml`.
2. Add `manifests/namespaces/billing.yaml` with `billing-dev`, `billing-qa` and `billing-prod`.
3. Commit and push. Argo CD creates the AppProject and the `billing-apps` app-of-apps.

**Domain team, new repo `billing-gitops`:** copy the layout of `payments-gitops`, replacing `payments` with `billing` in names, namespaces, repo URLs and the Kargo project.

**Credentials:** add `billing` to `DOMAINS` in `platform-bootstrap/scripts/bootstrap.sh` and include the new repo in the token's repository access, or register the repo and Kargo credentials by hand the same way the script does.

## Troubleshooting

| Symptom | Cause |
|---|---|
| `authorization failed: Write access to repository not granted` | The token lacks **Contents** access to that repo (GitHub returns this for reads too). |
| `unable to resolve 'stage/qa' to a commit SHA` | No promotion into that stage yet. Promote Freight into it in Kargo. |
| `... do not match any of the allowed destinations in project ...` | The AppProject is missing a destination; compare with `payments-project`. |
| App `Synced` but `WebApi` not ready | Check `kubectl describe webapi -n <ns>`; the `Degraded` and `Progressing` conditions carry the reason. |

## Reference

- [Argo CD Projects](https://argo-cd.readthedocs.io/en/stable/user-guide/projects/)
- [Argo CD cluster bootstrapping (app-of-apps)](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
- [Kargo documentation](https://docs.kargo.io/)
