# hello-gke-gitops

Helm charts, environment values, and Argo CD Applications for three Go services on GKE. This repo is **where** and **how** they run. App code and images live in [hello-gke-services](https://github.com/kubeboiii/hello-gke-services).

Argo CD watches this git branch, renders Helm, applies to the `hello-gke` namespace. After bootstrap, app rollouts are git commits, not `kubectl edit`.

**Not for:** prod hardening, multi-env overlays, or auto registry-to-git promotion. First-sync lab.

| | |
|---|---|
| **Time** | ~2–3 h first run |
| **You need** | `gcloud`, `kubectl`, Docker, GCP billing |
| **You get** | Three Synced Argo apps, one tag-bump exercise, working Ingress curl |

## Architecture

```
GitHub (your fork of this repo)
        │
        ▼
Argo CD (argocd namespace)
        │
        ▼
GKE / hello-gke namespace
├── frontend  → GCE Ingress
├── api       → ClusterIP
└── worker    → ClusterIP (no Ingress)
```

Traffic: Ingress → frontend → api. Worker proves a third Deployment fits the same GitOps flow without a public route.

## Before you start

1. **Fork this repo.** Argo reads GitHub, not your laptop.
2. Build and push images from `hello-gke-services` (tag `dev` or your choice).
3. In your fork: set `repoURL` in `argocd/applications/*-dev.yaml` (comment on that line), replace `my-gcp-project` in `environments/dev/*-values.yaml`, match `image.tag` to what you pushed.
4. **Commit and push to `main` on your fork** before expecting Argo to sync.

```bash
git add environments/dev argocd/applications
git commit -m "chore: set project ID and fork repoURL"
git push origin main
```

## Quick start

### 1. GKE cluster

```bash
gcloud config set project my-gcp-project

gcloud container clusters create hello-gke \
  --zone us-central1-a \
  --num-nodes 2 \
  --machine-type e2-medium

gcloud container clusters get-credentials hello-gke --zone us-central1-a
```

macOS + Homebrew `gcloud`: if `kubectl` fails auth, `source "$(brew --prefix)/share/google-cloud-sdk/path.zsh.inc"`.

### 2. Artifact Registry IAM

If pods sit in `ImagePullBackOff`, grant the compute default SA read access:

```bash
gcloud artifacts repositories add-iam-policy-binding hello-gke \
  --location=us-central1 \
  --member="serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

### 3. Install Argo CD

[bootstrap/argocd/install.md](bootstrap/argocd/install.md). Port-forward or LoadBalancer for the UI. Lock down anything public-facing when you are done playing.

### 4. Apply Applications

From your fork clone, after push to `main`:

```bash
kubectl apply -f argocd/projects/hello-gke.yaml
kubectl apply -f argocd/applications/
```

Creates `api-dev`, `frontend-dev`, `worker-dev`. Automated sync uses `prune: true` and `selfHeal: true`. Manual `kubectl` edits get reverted.

`valueFiles` paths are relative to the chart (`apps/api/helm`), not the Application file. Use `../../../environments/dev/api-values.yaml`, not `../../...`.

Optional App-of-Apps: `kubectl apply -f bootstrap/argocd/root-app.yaml`.

### 5. Verify

```bash
kubectl get applications -n argocd
kubectl get pods -n hello-gke
kubectl get ingress -n hello-gke
```

Ingress external IP can take a few minutes.

**GCE Ingress needs a Host header.** Bare IP curl returns 404 even when pods are fine.

```bash
curl -s -H "Host: hello-gke.dev.example.com" http://<INGRESS_IP>/ | jq
```

Host is set in `environments/dev/frontend-values.yaml`.

## GitOps image update

1. Change api code in `hello-gke-services`, build with `--platform linux/amd64`, push `api:v2`
2. Set `tag: v2` in `environments/dev/api-values.yaml`, commit, push your fork
3. Watch `api-dev` sync. Roll back with `git revert` or set `tag: dev`

```bash
kubectl patch application api-dev -n argocd \
  -p '{"metadata":{"annotations":{"argocd.argoproj.io/refresh":"hard"}}}' \
  --type merge
```

## Layout

```
apps/<service>/helm/       # Charts (Deployment, Service, Ingress on frontend)
environments/dev/          # Image tags, ingress host, replica counts
argocd/applications/       # Application CRDs
argocd/projects/           # hello-gke AppProject
bootstrap/argocd/          # Install notes + optional root app
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|----------------|-----|
| Argo sync error, values path not found | Wrong `valueFiles` depth (`../../` vs `../../../`) | Fix all three Application files, push |
| `ImagePullBackOff` | GAR IAM or wrong tag in values | IAM binding above; tag must exist in registry |
| `exec format error` | arm64 image on amd64 nodes | `docker build --platform linux/amd64`, use a new tag |
| `curl` to Ingress IP → 404 | Missing Host header | `curl -H "Host: hello-gke.dev.example.com" ...` |
| Argo UI stale after push | Cache | Hard refresh annotation above |
| Frontend 502 | api pod down | `kubectl logs -n hello-gke deploy/api` |

Hit something else? [Open an issue](https://github.com/kubeboiii/hello-gke-gitops/issues) with symptom and `kubectl describe pod` events.

## Cleanup

```bash
gcloud container clusters delete hello-gke --zone us-central1-a
```

A two-node `e2-medium` cluster left running will show up on your bill. Delete it when the lab is over.

## Related

- App repo: [hello-gke-services](https://github.com/kubeboiii/hello-gke-services)
- Extra drills: [LEARNING_PATH.md](LEARNING_PATH.md) (scale, probes, rollback)
