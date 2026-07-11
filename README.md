# hello-gke-gitops

GitOps manifests for deploying three Go microservices to GKE with Argo CD.

## Architecture

```
GitHub (this repo)
    │
    ▼
Argo CD (watches repo)
    │
    ▼
GKE cluster (hello-gke namespace)
    ├── frontend (Ingress exposed)
    ├── api (ClusterIP)
    └── worker (ClusterIP)
```

## Prerequisites

- `gcloud`, `kubectl`, `helm`, `argocd` CLI
- GCP project with billing enabled
- Container images pushed to Artifact Registry (see `hello-gke-services` repo)

## Step-by-step lab guide

### 1. Create GKE cluster

Replace `my-gcp-project` with your project ID.

```bash
gcloud config set project my-gcp-project

gcloud container clusters create hello-gke \
  --zone us-central1-a \
  --num-nodes 2 \
  --machine-type e2-medium \
  --workload-pool=my-gcp-project.svc.id.goog

gcloud container clusters get-credentials hello-gke --zone us-central1-a
```

### 2. Create Artifact Registry (if not done)

```bash
gcloud artifacts repositories create hello-gke \
  --repository-format=docker \
  --location=us-central1
```

Build and push images from the `hello-gke-services` repo, then update image tags in `environments/dev/*-values.yaml`.

### 3. Install Argo CD

Follow [bootstrap/argocd/install.md](bootstrap/argocd/install.md).

### 4. Configure repo URLs

Update `repoURL` in:

- `argocd/applications/*.yaml`
- `bootstrap/argocd/root-app.yaml`

Replace `YOUR_ORG` with your GitHub org or username.

### 5. Deploy applications

```bash
kubectl apply -f argocd/projects/hello-gke.yaml
kubectl apply -f argocd/applications/
```

Or use App-of-Apps:

```bash
kubectl apply -f bootstrap/argocd/root-app.yaml
```

### 6. Verify deployment

```bash
kubectl get applications -n argocd
kubectl get pods -n hello-gke
kubectl get ingress -n hello-gke
```

Get the external IP (may take a few minutes):

```bash
kubectl get ingress frontend -n hello-gke -w
```

Test:

```bash
curl http://<EXTERNAL_IP>/
```

### 7. Trigger a GitOps update

1. Change `image.tag` in `environments/dev/api-values.yaml`
2. Commit and push to `main`
3. Watch Argo CD auto-sync:

```bash
argocd app get api-dev
kubectl rollout status deployment/api -n hello-gke
```

## Repository layout

```
hello-gke-gitops/
├── apps/                    # Helm charts per service
├── environments/dev/        # Environment-specific values
├── argocd/
│   ├── applications/        # Argo CD Application CRDs
│   └── projects/            # Argo CD AppProject
├── bootstrap/argocd/        # Argo CD install + root app
└── LEARNING_PATH.md         # Hands-on exercises
```

## Troubleshooting

| Symptom | Check |
|---------|-------|
| `ImagePullBackOff` | Image exists in Artifact Registry; tag matches values file; GKE nodes can pull from GAR |
| App `OutOfSync` | Run `argocd app diff <app-name>`; check Helm value file paths |
| Ingress has no IP | Wait 5–10 min; verify GCE ingress controller; check `kubectl describe ingress` |
| Probe failures | `kubectl logs -n hello-gke deploy/<service>`; verify `/health` responds |
| Frontend 502 | API pod must be running; check `API_URL` env var |

## Cleanup

```bash
gcloud container clusters delete hello-gke --zone us-central1-a
```

## Next steps

- Workload Identity for GAR image pull (no imagePullSecrets)
- Argo CD Image Updater for automatic tag bumps
- Staging environment overlay in `environments/staging/`
- External Secrets Operator with GCP Secret Manager

## Related repo

Application source code: `hello-gke-services`
