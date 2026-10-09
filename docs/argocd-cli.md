# ArgoCD CLI cheatsheet

`argocd` is pinned in `.tool-versions`. It does what the web UI does, from
the terminal: inspect apps, diff against git, trigger syncs, read logs.

## Logging in

Pick one:

```sh
# Through the public hostname. --grpc-web is needed because Traefik sits in
# front of argocd-server.
argocd login argocd.marc-lab.dev --grpc-web --username admin

# No server at all: talks to the Kubernetes API with your kubeconfig.
# Simplest option if kubectl already works. No ArgoCD user/RBAC involved.
kubectl config set-context --current --namespace=argocd
argocd login --core

# Port-forward to argocd-server, if the hostname is unreachable.
argocd login --port-forward --port-forward-namespace argocd --plaintext
```

The initial admin password (until you change it and delete the Secret):

```sh
argocd admin initial-password -n argocd
argocd account update-password
```

Check who you are and which server you're on:

```sh
argocd account get-user-info
argocd context              # list/switch saved logins
```

## Looking at apps

```sh
argocd app list                          # all apps, sync + health status
argocd app get fastapi-playground        # details + every resource's status
argocd app get fastapi-playground --refresh       # re-read git first
argocd app get fastapi-playground --hard-refresh  # also drop the manifest cache
argocd app resources fastapi-playground  # flat list of managed resources
argocd app manifests fastapi-playground  # rendered manifests ArgoCD will apply
argocd app diff fastapi-playground       # live cluster vs. git
argocd app history fastapi-playground    # past syncs and their revisions
```

`--hard-refresh` is the one to use after rotating the GitHub token (see the
README), when an app is stuck on a repo auth error.

## Syncing

Every app here has `automated` sync with `prune` and `selfHeal`, so pushing
to git is normally all it takes (ArgoCD polls every ~3 minutes). The CLI is
for not waiting, or for watching it happen:

```sh
argocd app sync fastapi-playground                  # sync now
argocd app sync fastapi-playground --prune          # also delete removed resources
argocd app sync fastapi-playground --resource apps:Deployment:fastapi-playground
argocd app wait fastapi-playground --health --sync  # block until Synced + Healthy
argocd app sync root                                # pick up added/removed apps/ files
argocd app terminate-op fastapi-playground          # cancel a stuck sync
```

## Debugging

```sh
argocd app logs fastapi-playground --follow           # pod logs, all pods
argocd app logs fastapi-playground --kind Deployment --name fastapi-playground
argocd app actions list fastapi-playground --kind Deployment
argocd app actions run fastapi-playground restart --kind Deployment \
  --resource-name fastapi-playground                  # rollout restart
```

## Repos and credentials

```sh
argocd repo list        # repos ArgoCD has connected to, with status
argocd repocreds list   # credential templates (github-marcmarina-creds)
argocd cluster list     # just in-cluster here
argocd proj list        # just default here
```

## Things that won't stick here

The repo is the source of truth and `selfHeal` reverts drift, so a few
commands are pointless or refused on this cluster:

- **`argocd app set` / `app create` / `app delete`**: root manages every
  Application from `apps/`, so edits are reverted, created apps aren't in
  git, and deleted apps come back on the next sync. Edit the YAML and push.
- **`argocd app rollback`**: refused while automated sync is on. Revert the
  commit instead (`git revert`), or turn off `automated` in git first.
- **`argocd app patch`**: same as `app set`, reverted.

## Handy flags

- `-o yaml` / `-o json` / `-o wide` on most `get`/`list` commands.
- `argocd app list -l <label>` / `--project default` to filter.
- `argocd <command> --help` for everything else; `argocd completion zsh`
  for tab completion (`source <(argocd completion zsh)` in `.zshrc`).
