# How this repo works

```
k3s-cluster/
├── apps/
│   ├── root.yaml         ← applied by hand once, then syncs itself
│   ├── argocd.yaml       ← synced by root
│   ├── fastapi-playground.yaml
│   ├── go-playground.yaml
│   ├── hono-playground.yaml
│   └── turbo-express.yaml
├── values/argocd.yaml    ← read by apps/argocd.yaml
└── README.md
```

## `apps/root.yaml`: the entry point

The root of the "app of apps": an ArgoCD Application whose only job is to
sync other Application manifests.

- **`source.path: apps`**: watches the `apps/` folder of this repo on `main`.
  Every YAML file there gets applied to the cluster. Because those files are
  themselves Applications, each one becomes an app in ArgoCD.
- **It manages itself**: `root.yaml` lives in `apps/`, so root syncs its own
  manifest like any other. Something still has to start the chain:
  `kubectl apply -f apps/root.yaml` is run once on a fresh cluster, and from
  then on root comes from git too.
- **`destination.namespace: argocd`**: Application resources have to live in
  the `argocd` namespace. This doesn't control where the apps' own workloads
  go; each Application sets that itself.
- **`automated.selfHeal: true`**: if someone edits an Application by hand
  (with `kubectl edit` or in the ArgoCD UI), ArgoCD reverts it to match git.
  Root included: to change any sync policy, edit the file and push.
- **`prune: true`**: deleting a file from `apps/` (or moving it out, e.g.
  to an `archived/` folder) deletes the Application from the cluster. Apps
  with the `resources-finalizer` then take their workloads with them. Argo CD refuses
  to prune everything at once, so an empty `apps/` won't wipe the cluster.
- **No `resources-finalizer`**: if root itself is deleted or pruned, the
  child Applications and workloads keep running; nothing syncs until
  `apps/root.yaml` is applied again. Same recovery if a bad change to
  `root.yaml` breaks root: fix the file and `kubectl apply` it.

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
- **`selfHeal: true`, `prune: false`**: manual changes get reverted, but
  resources dropped from the chart are never deleted automatically. Being
  conservative here because they're ArgoCD's own.
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
- **`prune: true`**: unlike `argocd.yaml`, resources removed from the chart
  are deleted from the cluster.
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

## `apps/go-playground.yaml`: the Go API

Same pattern as `fastapi-playground.yaml`: the chart (`helm/` on `main`) and
the image tag (`values-image.yaml`) live in the `go-playground` repo, with
`prune: true` and the `resources-finalizer`. No `releaseName`, and the chart
sets `namespace:` from `.Values.namespace` (`default`). Served at
`go.marc-lab.dev` through Traefik's `web` entrypoint.

Unlike the other apps, it was added through this repo from the start
(2026-10-06) rather than created by hand and adopted.

## `apps/hono-playground.yaml`: the Hono API

Same pattern as `go-playground.yaml`: the chart (`helm/` on `main`) and the
image tag (`values-image.yaml`) live in the `hono-playground` repo, with
`prune: true` and the `resources-finalizer`. Served at `hono.marc-lab.dev`
through Traefik's `web` entrypoint.

Unlike the other apps, its chart also runs its own Postgres: a
`hono-playground-db` StatefulSet with a 1Gi volume from k3s's default
`local-path` StorageClass. Migrations run in an initContainer before the app
starts.

- **Password Secret, created by hand**: both Postgres and the app read the
  password from the `hono-playground-db` Secret, which isn't in git. Create
  it before the first sync (and again on a fresh cluster), or the pods stay
  in `CreateContainerConfigError`:

  ```sh
  kubectl -n default create secret generic hono-playground-db \
    --from-literal=password="$(openssl rand -hex 24)"
  ```

  Postgres only reads it when initialising an empty volume, so changing the
  Secret later doesn't change the database password.
- **Data outlives the app**: PVCs from a StatefulSet's `volumeClaimTemplates`
  aren't deleted with it, so removing the Application leaves
  `data-hono-playground-db-0` behind. Delete it by hand to wipe the data.

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
   it can manage anything. The chart version is read from `apps/argocd.yaml`
   with `awk`, so the install matches git and ArgoCD doesn't upgrade or
   downgrade itself right after bootstrap.
2. `kubectl apply -f apps/root.yaml`.
3. From there, root syncs `apps/`: it adopts its own manifest, and
   `apps/argocd.yaml` adopts the Helm install ArgoCD was started from. That's
   the same takeover that was done on the live cluster when this repo was set
   up (2026-10-05). The Helm release record stays behind and should be left
   alone.

It also has the command for joining extra nodes. That happens at the k3s
level and doesn't change anything in this repo.

Nothing reads the README automatically: root only syncs `apps/`, so it's
instructions for a person, not config.

## Not in this repo yet

- **Monitoring**: `kube-prometheus-stack` was removed on 2026-10-05, CRDs
  and data included, to be reinstalled from scratch. The apps' charts no
  longer ship a `ServiceMonitor`; add them back once the CRDs return.
- **Traefik**: installed and managed by k3s itself.
