# rollback-dev

Minimal nginx demo deployed by Argo CD and Argo Rollouts to Docker Desktop Kubernetes.

- Git branch: `dev`
- Application manifests: `k8s/demo`
- Kubernetes namespace: `rollback-dev`
- Argo CD application: `argocd/application.yaml`

## Updating the site

Edit `site/index.html` (or add CSS, JavaScript, and other static files under
`site/`), commit, and push to `dev`. GitHub Actions builds the Dockerfile,
publishes `ghcr.io/trovin/rollback-dev`, and commits its immutable image digest
to the Rollout manifest. Argo CD then deploys it automatically, stopping at
the configured 25% canary pause until you promote it.

The first publication creates a private GHCR package. Open
https://github.com/users/Trovin/packages/container/rollback-dev/settings and
change its visibility to Public so Kubernetes can pull it without credentials.
Then rerun the failed workflow from the Actions tab. Until anonymous pulling
succeeds, the workflow leaves the live image unchanged.

The workflow uses GitHub's built-in token with `contents: write` and
`packages: write`; no personal token or Kubernetes credentials are stored in CI.
Repository rules must allow the Actions bot to push to `dev`. The workflow's
manifest-only commit does not trigger another build. A manual run is available
after the workflow is present in the repository's default branch.

To roll back a site release without immediately rebuilding it, revert the
bot's image-update commit and push. The digest must still exist in GHCR.

Bootstrap after installing Argo CD:

```powershell
kubectl --context docker-desktop apply -f argocd/application.yaml
```

Argo CD automatically deploys commits to `dev` and repairs cluster drift.
Automatic deletion of resources removed from Git is disabled.

The demo uses an Argo Rollout with four replicas. The first deployment goes
directly to a healthy baseline. Later pod-template changes deploy a 25% canary
and pause for manual promotion before proceeding to 100%. Without a traffic
router, the weight is approximated by replica counts, not exact request routing.
The Rollout manifest is in `k8s/demo/deployment.yaml`.

If the Rollouts controller v1.10.2 is already installed but its Rollout or AnalysisRun CRDs are missing:

```powershell
kubectl --context docker-desktop apply --server-side -f https://raw.githubusercontent.com/argoproj/argo-rollouts/v1.10.2/manifests/crds/rollout-crd.yaml
kubectl --context docker-desktop apply --server-side -f https://raw.githubusercontent.com/argoproj/argo-rollouts/v1.10.2/manifests/crds/analysis-run-crd.yaml
kubectl --context docker-desktop -n argo-rollouts rollout restart deployment/argo-rollouts
```

Inspect the rollout and its pods:

```powershell
kubectl --context docker-desktop -n rollback-dev get rollouts,pods
kubectl --context docker-desktop -n rollback-dev describe rollout demo
```

To deploy an update, change the nginx image in Git, commit, and push to `dev`.
Argo CD syncs it automatically. Promote a paused canary using the Argo CD
resource action `promote` on the Rollout, or, with the kubectl Argo Rollouts plugin:

```powershell
kubectl argo rollouts --context docker-desktop -n rollback-dev promote demo
```

To roll back persistently, revert the image-change commit and push to `dev`.
Argo CD self-healing can overwrite direct changes to the live image.

Access Argo CD at https://localhost:8080 while this command is running:

```powershell
kubectl --context docker-desktop -n argocd port-forward svc/argocd-server 8080:443
```

Access the demo at http://localhost:8081 while this command is running:

```powershell
kubectl --context docker-desktop -n rollback-dev port-forward svc/demo 8081:80
```
