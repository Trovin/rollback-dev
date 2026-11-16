# rollback-dev

Minimal nginx demo deployed by Argo CD to Docker Desktop Kubernetes.

- Git branch: `dev`
- Application manifests: `k8s/demo`
- Kubernetes namespace: `rollback-dev`
- Argo CD application: `argocd/application.yaml`

Bootstrap after installing Argo CD:

```powershell
kubectl --context docker-desktop apply -f argocd/application.yaml
```

Argo CD automatically deploys commits to `dev` and repairs cluster drift.
Automatic deletion of resources removed from Git is disabled.

Access Argo CD at https://localhost:8080 while this command is running:

```powershell
kubectl --context docker-desktop -n argocd port-forward svc/argocd-server 8080:443
```

Access the demo at http://localhost:8081 while this command is running:

```powershell
kubectl --context docker-desktop -n rollback-dev port-forward svc/demo 8081:80
```
