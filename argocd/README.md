# ArgoCD on k3d (via Traefik)

Install:

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose over HTTP (no TLS at the Traefik edge) and restart the server:

```bash
kubectl apply -f cmd-params-patch.yaml
kubectl rollout restart deployment/argocd-server -n argocd
```

Route via Traefik ingress:

```bash
kubectl apply -f ingress.yaml
```

UI: http://argocd.195.201.121.117.nip.io
