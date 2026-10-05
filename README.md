# k3s-cluster

GitOps config for the `k3s-server` cluster, managed by ArgoCD.

```
apps/                 One ArgoCD Application per component (root included)
archived/             Applications no longer deployed
values/               Helm values referenced by the Applications
```

## Bootstrap a fresh cluster

Needs `helm`, `kubectl` and [`yq`](https://github.com/mikefarah/yq). The
chart version is read from `apps/argocd.yaml` so it can't drift from git.

```sh
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace \
  --version "$(yq '.spec.sources[0].targetRevision' apps/argocd.yaml)" \
  -f values/argocd.yaml
kubectl apply -f apps/root.yaml
```

From then on ArgoCD manages itself and root: change `values/argocd.yaml` or
bump the chart version in `apps/argocd.yaml`, push, and it syncs. Leave the
original Helm release alone (no `helm upgrade`/`helm uninstall`).

## Adding a node

Joining nodes happens at the k3s level; nothing in this repo changes. On the
new machine, with the token from `/var/lib/rancher/k3s/server/node-token` on
the server:

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 K3S_TOKEN=<token> sh -
```

This adds an agent (capacity only). The server is still the only control
plane node.
