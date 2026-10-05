# How this repo works

```
k3s-cluster/
├── bootstrap/root.yaml   ← applied by hand once
├── apps/
│   ├── argocd.yaml       ← synced by root
│   ├── fastapi-playground.yaml
│   ├── turbo-express.yaml
│   └── turbo-express-previews.yaml
├── values/argocd.yaml    ← read by apps/argocd.yaml
└── README.md
```

## `bootstrap/root.yaml`: the entry point

The root of the "app of apps": an ArgoCD Application whose only job is to
sync other Application manifests.

- **`source.path: apps`**: watches the `apps/` folder of this repo on `main`.
  Every YAML file there gets applied to the cluster. Because those files are
  themselves Applications, each one becomes an app in ArgoCD.
- **`destination.namespace: argocd`**: Application resources have to live in
  the `argocd` namespace. This doesn't control where the apps' own workloads
  go; each Application sets that itself.
- **`automated.selfHeal: true`**: if someone edits an Application by hand
  (for example with `kubectl edit`), ArgoCD reverts it to match git.
- **`prune: false`**: deleting a file from `apps/` does not delete the
  Application from the cluster; it just shows as out of sync. Deliberately
  conservative.
- **Why it's in `bootstrap/` and not `apps/`**: something has to start the
  chain, and it can't be ArgoCD itself. `kubectl apply -f bootstrap/root.yaml`
  is run once, and from then on everything else comes from git.

## `apps/argocd.yaml`: ArgoCD managing itself

This Application installs the ArgoCD Helm chart, which is what runs ArgoCD.

- **Two `sources`** (ArgoCD's multi-source feature):
  1. **The chart**: `argo-cd` from the official `argo-helm` repo, pinned to a
     chart version (`targetRevision`).
  2. **This repo, with `ref: values`**: doesn't deploy anything. It just gives
     the repo the name `$values` so the first source can read files from it.
- **`valueFiles: [$values/values/argocd.yaml]`**: render the upstream chart
  using the values file from this repo. Keeping the values separate means the
  chart itself never has to be copied into the repo.
- **`releaseName: argocd`**: must match the original Helm release name.
  Resource names are built from it (`argocd-server`, `argocd-repo-server`, …),
  and a different name would create a second set of resources instead of
  adopting the existing ones.
- **`destination.namespace: argocd`**: where the chart's resources go.
- **`ServerSideApply=true`**: needed because the ApplicationSet CRD is too big
  for regular client-side apply. The original Helm install also used
  server-side apply, so behaviour stays the same.
- **`selfHeal: true`, `prune: false`**: same reasoning as root. Manual changes
  get reverted; nothing gets deleted automatically.
- **No `resources-finalizer`**: normally that finalizer makes deleting an
  Application also delete everything it deployed. Here that would mean
  deleting the Application deletes ArgoCD itself, so it's left off on purpose.

**Upgrading ArgoCD:** bump the chart's `targetRevision`, commit and push.
ArgoCD picks up the change and upgrades itself.

## `apps/turbo-express.yaml`: the Express app

Deploys the Express app from the `turbo-playground` monorepo. Unlike
`argocd.yaml`, the chart and its values live in the app's own repo
(`apps/express/helm` on `master`), not here:

- **`values.yaml` + `values-image.yaml`**: both come from `turbo-playground`.
  The image tag lives in `values-image.yaml`, so deploying a new version means
  changing that file in the app repo, not this one.
- **`releaseName: turbo-express`**: keeps resource names the same as when the
  app was installed with Helm.
- **`prune: true`**: unlike the platform apps, resources removed from the
  chart are deleted from the cluster.
- **`resources-finalizer`**: deleting this Application also deletes the app's
  Deployment, Service, etc. That's the normal behaviour for an app (the
  opposite of `argocd.yaml`, where it would delete ArgoCD itself).

The Application was created by hand first and moved here on 2026-10-05; the
file matches what was live exactly, so root adopted it without changing it.

## `apps/fastapi-playground.yaml`: the FastAPI app

Same pattern as `turbo-express.yaml`: the chart (`helm/` on `main`) and the
image tag (`values-image.yaml`) live in the `fastapi-playground` repo, with
`prune: true` and the `resources-finalizer`.

- **No `releaseName`**: it defaults to the Application name,
  `fastapi-playground`.
- **Namespace**: the chart's templates set `namespace:` from
  `.Values.namespace` (`default`), so the chart decides where resources go.
  `destination.namespace` only applies to resources that don't set one.

## `apps/turbo-express-previews.yaml`: PR previews

An ApplicationSet, not an Application: its Pull Request generator polls
`turbo-playground` every 5 minutes and creates one Application per open PR
labelled `preview`.

- **Per-PR names**: Application, namespace and hostname are all
  `turbo-express-pr-<number>` (`turbo-express-pr-<number>.marc-lab.dev`).
  The `*.marc-lab.dev` wildcard DNS already covers the hostname.
- **`targetRevision` and `image.tag` set to the PR's head SHA**: the chart is
  rendered from the PR's commit, and the image is the one the `preview.yml`
  workflow in `turbo-playground` builds for that commit. `values-image.yaml`
  is skipped, since it holds the tag for production.
- **Images can lag behind**: ArgoCD may deploy a commit before its image is
  pushed, which shows as `ImagePullBackOff` until the build finishes and the
  pull is retried.
- **No GitHub token**: the repo is public, and polling every 5 minutes stays
  well under the unauthenticated API limit.
- **Cleanup**: closing or merging the PR removes the Application, and the
  `resources-finalizer` deletes what it deployed. The namespace made by
  `CreateNamespace=true` is left behind and has to be deleted by hand.

## `values/argocd.yaml`: ArgoCD's configuration

The Helm values originally passed to `helm install`, copied unchanged (the
chart renders identically with this file and with the original live values).

- **`global.domain: argocd.marc-lab.dev`**: the hostname the chart uses for
  the ingress and for ArgoCD's own URL setting (`url` in `argocd-cm`).
- **`configs.params.server.insecure: true`**: argocd-server serves plain HTTP.
  HTTPS is handled in front of it, so ArgoCD doesn't need its own certificate.
  The ingress only uses Traefik's `web` (port 80) entrypoint, so TLS is
  likely terminated by something in front of Traefik rather than by Traefik.
- **`controller`, `repoServer`, `applicationSet` `replicas: 1`**: one copy of
  each component, which suits a single-node cluster.
- **`redis-ha.enabled: false`**: a single Redis instance instead of the
  high-availability Redis cluster.
- **`server.ingress`**: creates an Ingress for the UI using the `traefik`
  ingress class, on Traefik's `web` entrypoint.

Changing ArgoCD's config works the same way as upgrading: edit this file and
push. For example, notification settings go under `notifications:` and SSO
under `configs.cm`. Any value the chart supports can go here; to see the full
list for the pinned version:

```sh
helm show values argo/argo-cd --version <chart version>
```

## `README.md`

A short description of the layout and the steps to rebuild the cluster from
scratch:

1. `helm install` ArgoCD with `values/argocd.yaml`. ArgoCD has to exist before
   it can manage anything.
2. `kubectl apply -f bootstrap/root.yaml`.
3. From there, root syncs `apps/argocd.yaml`, and ArgoCD adopts the Helm
   install it was started from. That's the same takeover that was done on the
   live cluster when this repo was set up (2026-10-05).

## Not in this repo yet

- **Monitoring**: `kube-prometheus-stack` was removed on 2026-10-05, CRDs
  and data included, to be reinstalled from scratch. The apps' charts no
  longer ship a `ServiceMonitor`; add them back once the CRDs return.
- **Traefik**: installed and managed by k3s itself.
