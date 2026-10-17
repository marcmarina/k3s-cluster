# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

GitOps config for a single-server k3s cluster, managed by ArgoCD. There is no build or test step: pushing to `main` is the deploy (ArgoCD polls every ~3 minutes). `README.md` covers bootstrapping a cluster, private-repo credentials and adding nodes; `docs/how-it-works.md` explains every Application field by field; `docs/argocd-cli.md` is the `argocd` CLI cheatsheet (CLI version pinned in `.tool-versions`).

## Architecture

- `apps/root.yaml` is the app of apps. It syncs `apps/` recursively but only files matching `{root.yaml,*/application.yaml}`, and it manages (and self-heals) itself. Anything else in an app folder is ignored by root, so a plain manifest needs its own Application.
- `apps/<name>/application.yaml` is one ArgoCD Application per component. Two patterns:
  - **Upstream charts** (`argocd`, `kube-prometheus-stack`): multi-source — the chart pinned by `targetRevision`, plus this repo as `ref: values`, read via `valueFiles: [$values/apps/<name>/values.yaml]`. Upgrade by bumping the chart's `targetRevision`.
  - **Own apps** (`*-playground`, `turbo-express`, `turbo-koa`, `mintlify`): the Helm chart and the image tag (`values-image.yaml`) live in the app's own GitHub repo. Deploying a new version happens there, not here.
- `archived/` has the same layout but is outside `apps/`, so it isn't synced. Archive/restore with `git mv apps/<name> archived/` (and back); if the folder has a `values.yaml`, fix the `$values/...` path in its `application.yaml` in the same commit.

## Invariants to preserve

- Every Application sets `metadata.namespace: argocd`; the workload namespace is set in `destination` (or by the chart itself).
- All apps use automated sync with `selfHeal`, so `kubectl edit`, `argocd app set/patch/create/delete` and `argocd app rollback` don't stick. Change the YAML and push; roll back with `git revert`.
- `root` and `argocd` deliberately have **no** `resources-finalizer` (deleting them must not cascade to children / ArgoCD itself). `argocd` also has `prune: false`. Regular apps do have the finalizer and `prune: true`.
- In `apps/argocd/application.yaml` the chart source must stay first: the README bootstrap reads the first `targetRevision:` with `awk`. `releaseName: argocd` must match the original Helm release; never `helm upgrade`/`uninstall` that release.
- `ServerSideApply=true` is required where charts ship large CRDs (ArgoCD, kube-prometheus-stack).
- Secrets are never committed. Private repos use a hand-created `repo-creds` Secret (`github-marcmarina-creds`), and some apps need hand-created Secrets (`hono-playground-db`, `drizzle-playground-dotenvx-keys`) — see the README and `docs/how-it-works.md`.
- Hosts are `*.marc-lab.dev` via Traefik's `web` entrypoint (Traefik is managed by k3s, not this repo). `prometheus.marc-lab.dev` has no auth and must stay LAN-only.

When adding, moving or changing an app, update `docs/how-it-works.md` (and the tree at its top) to match.

## Useful commands

```sh
# Render an upstream chart with this repo's values (version from application.yaml)
helm template argocd argo/argo-cd --version <ver> -f apps/argocd/values.yaml
helm show values argo/argo-cd --version <ver>

argocd app diff <app>            # live vs. git
argocd app sync root             # pick up added/removed apps without waiting
argocd app get <app> --hard-refresh
```
