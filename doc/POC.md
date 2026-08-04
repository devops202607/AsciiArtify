# POC — ArgoCD on k3d (via Traefik)

Install:

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose over HTTP (no TLS at the Traefik edge) and restart the server:

```bash
kubectl apply -f argocd/cmd-params-patch.yaml
kubectl rollout restart deployment/argocd-server -n argocd
```

Route via Traefik ingress (HTTPS with a self-signed cert):

```bash
# map port 443 into the cluster (once, per cluster)
k3d cluster edit k3d-demo --port-add "443:443@loadbalancer"

# generate a self-signed cert for the argocd host
openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout argocd-tls.key -out argocd-tls.crt \
  -subj "/CN=argocd.195.201.121.117.nip.io" \
  -addext "subjectAltName=DNS:argocd.195.201.121.117.nip.io"

kubectl create secret tls argocd-tls -n argocd \
  --cert=argocd-tls.crt --key=argocd-tls.key

kubectl apply -f argocd/ingress.yaml
```

Get the admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
```

UI: https://argocd.195.201.121.117.nip.io (accept the self-signed cert)

## Demostration

![ArgoCD demo](argocd.gif)

![CD demo](demo-cd.gif)
