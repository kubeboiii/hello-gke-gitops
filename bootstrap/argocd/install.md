# Install Argo CD

## Prerequisites

- GKE cluster running and `kubectl` configured
- Helm 3 installed

## Install

```bash
kubectl apply -f bootstrap/argocd/namespace.yaml

helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --set server.service.type=LoadBalancer
```

## Get admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

## Port-forward (alternative to LoadBalancer)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open https://localhost:8080 (user: `admin`).

## Bootstrap GitOps apps

1. Update `repoURL` in all files under `argocd/applications/` and `bootstrap/argocd/root-app.yaml`
2. Apply the AppProject:

```bash
kubectl apply -f argocd/projects/hello-gke.yaml
```

3. Either apply individual apps:

```bash
kubectl apply -f argocd/applications/
```

Or use the App-of-Apps pattern:

```bash
kubectl apply -f bootstrap/argocd/root-app.yaml
```

## Verify

```bash
kubectl get applications -n argocd
kubectl get pods -n hello-gke
argocd login <ARGOCD_SERVER> --username admin --password <PASSWORD>
argocd app list
```
