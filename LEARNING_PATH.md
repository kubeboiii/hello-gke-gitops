# Learning Path — Argo CD + GKE

Ordered exercises for hands-on GitOps learning. Complete them in sequence.

## Exercise 1: Deploy and verify

**Goal:** Get all three services running and hit the frontend via Ingress.

1. Push images from `hello-gke-services` to Artifact Registry
2. Update `environments/dev/*-values.yaml` with your project ID and image tag
3. Apply Argo CD Applications
4. Wait for all apps to show `Synced` and `Healthy` in Argo CD UI
5. Get Ingress IP and curl the frontend:

```bash
curl http://<INGRESS_IP>/
```

**Expected:** JSON response with `frontend` message and nested `api` payload.

**Learn:** Argo CD watches Git, renders Helm, applies to cluster. Ingress exposes only frontend.

---

## Exercise 2: GitOps image update

**Goal:** Change app behavior by updating Git, not kubectl.

1. In `hello-gke-services`, change the API response message in `services/api/main.go`
2. Rebuild and push image with a new tag (e.g. `v2`)
3. Update `environments/dev/api-values.yaml` with the new tag
4. Commit and push to `main`
5. Watch Argo CD sync automatically

```bash
argocd app get api-dev
kubectl rollout status deployment/api -n hello-gke
curl http://<INGRESS_IP>/
```

**Learn:** Git is the source of truth. Image tag changes flow through GitOps.

---

## Exercise 3: Scale via values

**Goal:** Change replica count through Git.

1. Set `replicaCount: 2` in `environments/dev/frontend-values.yaml`
2. Push to `main`
3. Verify:

```bash
kubectl get pods -n hello-gke -l app.kubernetes.io/name=frontend
```

**Learn:** Helm values in Git control runtime config. No `kubectl scale`.

---

## Exercise 4: Break and fix readiness probe

**Goal:** Understand how Argo CD and K8s handle unhealthy pods.

1. In `apps/api/helm/templates/deployment.yaml`, temporarily change readiness probe path to `/broken`
2. Push to `main`
3. Observe: pods not ready, Argo CD may show Degraded
4. Fix the probe path, push again
5. Watch recovery

**Learn:** Probes gate traffic. GitOps rollback is a Git revert or Argo CD history rollback.

---

## Exercise 5: Roll back with Argo CD

**Goal:** Use Argo CD history to undo a bad deploy.

1. Deploy a change you want to undo (e.g. Exercise 4 break)
2. In Argo CD UI or CLI, view history:

```bash
argocd app history api-dev
argocd app rollback api-dev <REVISION>
```

3. Or revert the Git commit and let auto-sync restore the previous state

**Learn:** Argo CD keeps deployment history. Rollback = previous known-good revision.

---

## Bonus exercises

- **App-of-Apps:** Delete individual apps, apply only `bootstrap/argocd/root-app.yaml`, watch child apps appear
- **Self-heal:** `kubectl delete deployment frontend -n hello-gke`, watch Argo CD recreate it
- **Manual drift:** `kubectl set image deployment/api api=wrong:tag -n hello-gke`, watch Argo CD revert on sync
- **Argo CD CLI sync:** `argocd app sync frontend-dev --force`

## Concepts checklist

After completing all exercises, you should understand:

- [ ] Git as single source of truth for cluster state
- [ ] Argo CD Application CRD (source, destination, syncPolicy)
- [ ] Helm charts + environment value files
- [ ] Automated sync with prune and selfHeal
- [ ] App-of-Apps bootstrap pattern
- [ ] GKE Ingress for external traffic
- [ ] Internal service discovery (`http://api:8081`)
- [ ] Image tag promotion via Git commits
