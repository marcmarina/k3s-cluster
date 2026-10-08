# k3s-cluster

GitOps config for the `k3s-server` cluster, managed by ArgoCD.

```
apps/                 One ArgoCD Application per component (root included)
values/               Helm values referenced by the Applications
```

## Bootstrap a fresh cluster

Needs `helm` and `kubectl`. The chart version is read from `apps/argocd.yaml`
(the first `targetRevision:` in the file, so keep the chart source first) so
it can't drift from git.

```sh
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace \
  --version "$(awk '/targetRevision:/ {print $2; exit}' apps/argocd.yaml)" \
  -f values/argocd.yaml
# Create the GitHub credentials here (see "Private repositories" below),
# or apps from private repos won't sync
kubectl apply -f apps/root.yaml
```

From then on ArgoCD manages itself and root: change `values/argocd.yaml` or
bump the chart version in `apps/argocd.yaml`, push, and it syncs. Leave the
original Helm release alone (no `helm upgrade`/`helm uninstall`).

## Private repositories

ArgoCD reads private repos with a credential template: a `repo-creds` Secret
whose `url` is a prefix, so every `repoURL` under
`https://github.com/marcmarina` uses it and no repo has to be registered on
its own. The Secret is created by hand and isn't in git, so it has to be
recreated on a fresh cluster.

Create a fine-grained personal access token (GitHub → Settings → Developer
settings) with read-only **Contents** access to the private repos, then:

```sh
kubectl -n argocd create secret generic github-marcmarina-creds \
  --from-literal=type=git \
  --from-literal=url=https://github.com/marcmarina \
  --from-literal=username=marcmarina \
  --from-literal=password=<token>
kubectl -n argocd label secret github-marcmarina-creds \
  argocd.argoproj.io/secret-type=repo-creds
```

The token expires (a year at most). To rotate it, delete the Secret and
create it again with the new token, then hard refresh any app showing an
auth error.

Don't put the token in `values/argocd.yaml` (`configs.credentialTemplates`):
that file is committed in plain text.

Alternatives, if this gets annoying:

- **GitHub App** instead of a token: not tied to a personal account, and
  ArgoCD fetches short-lived tokens itself, so nothing expires. Same Secret,
  with `githubAppID`, `githubAppInstallationID` and `githubAppPrivateKey`
  in place of `username`/`password`.
- **Sealed Secrets** to keep the Secret in git, encrypted, so a fresh cluster
  needs no manual step (only a backup of the controller's sealing key).
- **SSH deploy keys** work too, but GitHub allows one repo per key, so they
  don't scale past a couple of repos.

## Adding a node

Joining nodes happens at the k3s level; nothing in this repo changes. On the
new machine, with the token from `/var/lib/rancher/k3s/server/node-token` on
the server:

```sh
curl -sfL https://get.k3s.io | K3S_URL=https://<server-ip>:6443 K3S_TOKEN=<token> sh -
```

This adds an agent (capacity only). The server is still the only control
plane node.
