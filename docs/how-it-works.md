# How this repo works

```
k3s-cluster/
├── apps/
│   ├── root.yaml                  ← applied by hand once, then syncs itself
│   ├── argocd/
│   │   ├── application.yaml       ← synced by root
│   │   └── values.yaml            ← read by argocd/application.yaml
│   ├── kube-prometheus-stack/
│   │   ├── application.yaml
│   │   └── values.yaml
│   ├── drizzle-playground/application.yaml
│   ├── fastapi-playground/application.yaml
│   ├── hono-playground/application.yaml
│   ├── turbo-express/application.yaml
│   └── turbo-koa/application.yaml
├── archived/                      ← same layout, not synced (outside apps/)
│   ├── forgetful-api/application.yaml
│   └── go-playground/application.yaml
└── README.md
```

## `apps/root.yaml`: the entry point

The root of the "app of apps": an ArgoCD Application whose only job is to
sync other Application manifests.

- **`source.path: apps`** with **`directory.recurse: true`**: watches the
  `apps/` folder of this repo on `main`, subfolders included. Every YAML file
  there gets applied to the cluster. Because those files are themselves
  Applications, each one becomes an app in ArgoCD.
- **`directory.include: '{root.yaml,*/application.yaml}'`**: root only
  applies Applications. Anything else in a component folder is ignored,
  such as its Helm values (not Kubernetes manifests; the Application reads
  them through `$values`). `root.yaml` has to be in the pattern too, or root
  would stop managing (and prune) itself. A plain manifest next to an
  `application.yaml` would also be ignored, so it needs its own Application.
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
  to `archived/`) deletes the Application from the cluster. Apps with the
  `resources-finalizer` then take their workloads with them. Argo CD refuses
  to prune everything at once, so an empty `apps/` won't wipe the cluster.
- **Archiving**: `archived/` uses the same layout as `apps/`, so archiving
  or restoring an app is moving its whole folder:
  `git mv apps/<name> archived/` and back with `git mv archived/<name> apps/`.
  If the folder has a `values.yaml`, update the `$values/...` path in its
  `application.yaml` in the same commit.
- **No `resources-finalizer`**: if root itself is deleted or pruned, the
  child Applications and workloads keep running; nothing syncs until
  `apps/root.yaml` is applied again. Same recovery if a bad change to
  `root.yaml` breaks root: fix the file and `kubectl apply` it.

## `apps/argocd/application.yaml`: ArgoCD managing itself

This Application installs the ArgoCD Helm chart, which is what runs ArgoCD.

- **Two `sources`** (ArgoCD's multi-source feature):
  1. **The chart**: `argo-cd` from the official `argo-helm` repo, pinned to a
     chart version (`targetRevision`).
  2. **This repo, with `ref: values`**: doesn't deploy anything. It just gives
     the repo the name `$values` so the first source can read files from it.
- **`valueFiles: [$values/apps/argocd/values.yaml]`**: render the upstream
  chart using the values file from this repo. Keeping the values separate
  means the chart itself never has to be copied into the repo.
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

## `apps/turbo-express/application.yaml`: the Express app

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

## `apps/turbo-koa/application.yaml`: the Koa app

Same pattern as `turbo-express.yaml`, from the same `turbo-playground` repo
and the same image: the chart (`apps/koa/helm` on `master`) and the image tag
(`values-image.yaml`) live there, with `prune: true` and the
`resources-finalizer`. The app repo's CI bumps both apps' image tags in one
commit, so they always deploy the same build. Served at
`turbo-koa.marc-lab.dev` through Traefik's `web` entrypoint.

- **No `releaseName`**: it defaults to the Application name, `turbo-koa`.
- **No `ServiceMonitor`**: the Koa app has no `/metrics` endpoint.

Added through this repo from the start (2026-10-16).

## `apps/fastapi-playground/application.yaml`: the FastAPI app

Same pattern as `turbo-express.yaml`: the chart (`helm/` on `main`) and the
image tag (`values-image.yaml`) live in the `fastapi-playground` repo, with
`prune: true` and the `resources-finalizer`.

- **No `releaseName`**: it defaults to the Application name,
  `fastapi-playground`.
- **Namespace**: the chart's templates set `namespace:` from
  `.Values.namespace` (`default`), so the chart decides where resources go.
  `destination.namespace` only applies to resources that don't set one.

## `archived/go-playground/application.yaml`: the Go API

Same pattern as `fastapi-playground.yaml`: the chart (`helm/` on `main`) and
the image tag (`values-image.yaml`) live in the `go-playground` repo, with
`prune: true` and the `resources-finalizer`. No `releaseName`, and the chart
sets `namespace:` from `.Values.namespace` (`default`). Served at
`go.marc-lab.dev` through Traefik's `web` entrypoint.

Unlike the other apps, it was added through this repo from the start
(2026-10-06) rather than created by hand and adopted.

## `apps/hono-playground/application.yaml`: the Hono API

Same pattern as `go-playground.yaml`: the chart (`helm/` on `main`) and the
image tag (`values-image.yaml`) live in the `hono-playground` repo, with
`prune: true` and the `resources-finalizer`. Served at `hono.marc-lab.dev`
through Traefik's `web` entrypoint.

Its database lives outside the cluster: a Postgres LXC on the local network
at `192.168.31.143`. Migrations run in an initContainer before the app
starts.

- **Password Secret, created by hand**: host, port, database and user are
  plain env vars in the chart's `values.yaml`; only `DATABASE_PASSWORD` comes
  from the `password` key of the `hono-playground-db` Secret, which isn't in
  git. Create it before the first sync (and again on a fresh cluster), or the
  pod stays in `CreateContainerConfigError`:

  ```sh
  kubectl -n default create secret generic hono-playground-db \
    --from-literal=password='<password>'
  ```

## `apps/drizzle-playground/application.yaml`: the Express + Drizzle API

Same pattern as `hono-playground.yaml`: the chart (`helm/` on `main`) and the
image tag (`values-image.yaml`) live in the `drizzle-playground` repo, with
`prune: true` and the `resources-finalizer`. Served at
`drizzle-playground.marc-lab.dev` through Traefik's `web` entrypoint.

It uses the same Postgres LXC as hono (`192.168.31.143`, database and user
`drizzle`), and migrations run in an initContainer before the app starts.

- **Config from dotenvx, key in a Secret created by hand**: unlike hono, the
  app's config (database password included) lives encrypted in
  `environments/.env.production` in the app repo and is decrypted at
  startup. Only the private key comes from the
  `DOTENV_PRIVATE_KEY_PRODUCTION` key of the `drizzle-playground-dotenvx-keys`
  Secret, which isn't in git. Create it before the first sync (and again on a
  fresh cluster), or the pod stays in `CreateContainerConfigError`:

  ```sh
  kubectl -n default create secret generic drizzle-playground-dotenvx-keys \
    --from-literal=DOTENV_PRIVATE_KEY_PRODUCTION='<key>'
  ```

## `apps/kube-prometheus-stack/application.yaml`: monitoring

Prometheus, Alertmanager, node-exporter, kube-state-metrics and the
Prometheus Operator, in the `monitoring` namespace. Grafana is not part of
it (see below). Same two-source pattern
as `argocd.yaml`: the upstream chart, with values from
`apps/kube-prometheus-stack/values.yaml` in this repo. To upgrade, bump the chart's
`targetRevision`.

- **`CreateNamespace=true`**: ArgoCD creates `monitoring` on the first sync.
- **`ServerSideApply=true`**: the operator's CRDs (`Prometheus`,
  `ServiceMonitor`, ...) are too large for client-side apply, like ArgoCD's
  ApplicationSet CRD.
- **`ignoreDifferences` on the webhook `caBundle`**: a Job in the chart
  generates a self-signed certificate and patches it into the operator's
  admission webhooks after sync. Without this, the app would always show
  OutOfSync and self-heal would keep wiping the certificate.
- **`prune: true` and the `resources-finalizer`**: like the apps, deleting
  this Application removes everything, including the CRDs (and with them
  every `ServiceMonitor` in the cluster) and Prometheus' data volume.

It was added on 2026-10-12, reinstalled from scratch after the old
hand-installed release was removed on 2026-10-05.

## `apps/kube-prometheus-stack/values.yaml`: monitoring configuration

- **Control plane monitors off**: k3s runs the controller-manager,
  scheduler, kube-proxy and etcd inside the single `k3s` process rather than
  as separate pods, so the chart's ServiceMonitors for them have nothing to
  scrape and would fire `TargetDown` forever. Their alert rules are off too.
  The API server, kubelet and CoreDNS are still scraped.
- **`*SelectorNilUsesHelmValues: false`**: Prometheus picks up every
  `ServiceMonitor`, `PodMonitor`, `PrometheusRule`, `Probe` and
  `ScrapeConfig` in the cluster, so the apps' charts can ship monitors without
  a `release: kube-prometheus-stack` label.
- **Storage**: Prometheus keeps 15 days of data (capped at 8GB) on a 10Gi
  `local-path` volume, Alertmanager on 1Gi. `local-path` stores it on the
  node's disk, so the data is tied to that node.
- **Prometheus ingress**: served at `prometheus.marc-lab.dev` through
  Traefik's `web` entrypoint. Prometheus has no login, so this host is meant
  to be reached only through the local Caddy server, which keeps it on the
  LAN. Don't add it to public DNS or forward it from outside.
- **No Grafana**: the chart's bundled Grafana is disabled so Grafana can be
  run and upgraded separately. Point it at Prometheus at
  `http://kube-prometheus-stack-prometheus.monitoring.svc:9090` (from inside
  the cluster). `forceDeployDashboards: true` keeps the chart's dashboards as
  ConfigMaps labelled `grafana_dashboard: "1"`, so a Grafana in the cluster
  with the dashboard sidecar picks them up automatically.

## `apps/argocd/values.yaml`: ArgoCD's configuration

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

1. `helm install` ArgoCD with `apps/argocd/values.yaml`. ArgoCD has to exist
   before it can manage anything. The chart version is read from
   `apps/argocd/application.yaml` with `awk`, so the install matches git and
   ArgoCD doesn't upgrade or downgrade itself right after bootstrap.
2. `kubectl apply -f apps/root.yaml`.
3. From there, root syncs `apps/`: it adopts its own manifest, and
   `apps/argocd/application.yaml` adopts the Helm install ArgoCD was started
   from. That's the same takeover that was done on the live cluster when this
   repo was set up (2026-10-05). The Helm release record stays behind and should be left
   alone.

It also has the command for joining extra nodes. That happens at the k3s
level and doesn't change anything in this repo.

Nothing reads the README automatically: root only syncs `apps/`, so it's
instructions for a person, not config.

## Not in this repo yet

- **App `ServiceMonitor`s**: the apps' charts stopped shipping them when
  monitoring was removed on 2026-10-05. Now that the CRDs are back (see
  `apps/kube-prometheus-stack/`), they can be added again.
- **Traefik**: installed and managed by k3s itself.
