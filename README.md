# k3s-cluster

GitOps config for the `k3s-server` cluster, managed by ArgoCD.

```
bootstrap/root.yaml   App of apps, applied by hand once
apps/                 One ArgoCD Application per component
values/               Helm values referenced by the Applications
```

## Bootstrap a fresh cluster

```sh
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace --version 10.9.6 -f values/argocd.yaml
kubectl apply -f bootstrap/root.yaml
```

From then on ArgoCD manages itself: change `values/argocd.yaml` or bump the
chart version in `apps/argocd.yaml`, push, and it syncs.
