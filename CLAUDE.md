# CLAUDE.md: platform-gitops

Platform team GitOps repo, synced by Argo CD. It holds the two root app-of-apps targets and the platform docs.

## Layout

- `apps/infrastructure/`: platform Applications (namespaces, cert-manager, Kargo, Kargo RBAC, Grafana, webapi-operator per env, the operator's Kargo project). Ordered with sync waves. Target of the `infra-apps` root app.
- `apps/domains/<domain>/`: `project.yaml` (AppProject = tenant boundary: `<domain>-*` namespaces, own repo only, no cluster-scoped resources except its Kargo `Project`) and `apps.yaml` (domain app-of-apps pointing at `<domain>-gitops`). Target of the `domains-apps` root app (recurse).
- `manifests/`: Helm values and plain manifests referenced by `apps/`. `webapi-operator-{dev,qa,prod}/values.yaml` pin the operator image tag and namespace selector.
- `docs/`: `MULTI_REPO_SETUP.md`, plus architecture and decision docs as they land.

## Build, test, run

No build. Validate before pushing:

```bash
kubectl apply --dry-run=client -f apps/ -R              # Application objects parse
kustomize build manifests/<dir>                         # where a kustomization exists
helm template <chart> -f manifests/<app>/values.yaml    # for Helm-backed apps
```

Changes take effect when pushed to `main` (Argo CD polls about every 3 minutes; force with a hard-refresh annotation). Run the platform from `platform-bootstrap` (`make all`).

## Notes

- Demo-only credentials (Grafana `admin/admin`, Kargo admin hash and signing key in `manifests/kargo/values.yaml`) are documented as such. Never add real secrets.
- Adding a domain: `apps/domains/<name>/{project,apps}.yaml`, namespaces in `manifests/namespaces/`, a `<name>-gitops` repo, and the domain in `platform-bootstrap/scripts/bootstrap.sh` `DOMAINS`.

## The five-repo platform

| Repo | Role |
|---|---|
| `webapi-operator` | Go/kubebuilder operator. Owns the `WebApi` CRD (`platform.aaronstahley.dev/v1alpha1`) and reconciles it into a Deployment and Service. Ships a Helm chart and a GHCR image. |
| `platform-gitops` | Platform team repo. Argo CD app-of-apps (`apps/infrastructure`, `apps/domains`), AppProjects as tenant boundaries, namespaces, Kargo, operator instances per env, docs. |
| `platform-bootstrap` | Day-0: kind cluster, Argo CD install, credentials, root app-of-apps. `make all`. |
| `payments-gitops` | Domain repo: podinfo as a `WebApi`, Kustomize overlays per env, Kargo warehouse and stages. |
| `orders-gitops` | Same shape as payments, for the orders domain. |

Flow: a developer changes a `WebApi` in a domain repo on Git. Kargo turns the change into Freight and promotes dev -> qa -> prod by rendering into `stage/<env>` branches. Argo CD syncs each env from its branch, and the operator reconciles the `WebApi`.

## Conventions (all repos)

- Keep commits small and meaningful.
- Never hand-edit generated files (deepcopy, CRDs, RBAC). In webapi-operator run `make manifests generate`, then refresh the Helm chart's CRD copy in `helm/webapi-operator/crds/`.
- Demo workload is podinfo: `ghcr.io/stefanprodan/podinfo` `6.15.0`, port `9898`, liveness `/healthz`, readiness `/readyz`.
- Domains are `payments` and `orders`. CRD group/version is `platform.aaronstahley.dev/v1alpha1`.
- These repos are public. No secrets, tokens or personal email addresses.
- Run tests and the build after changes, and only commit/push when they pass.
