# GitOps Workflow with ArgoCD on Kubernetes

Sync Kubernetes deployment state directly from a Git repository using ArgoCD —
no `kubectl apply` needed after initial setup. Once configured, ArgoCD watches
this repo and automatically applies any change you push.

## Project structure

```
gitops-argocd-demo/
├── manifests/
│   ├── deployment.yaml       # The app ArgoCD deploys — this is the "source of truth"
│   └── service.yaml
├── argocd/
│   └── application.yaml      # Tells ArgoCD what repo/path to watch and auto-sync
├── NOTES.md                  # GitOps flow explanation (deliverable)
└── README.md
```

This repo pairs with the `cicd-demo` CI/CD repo from the previous project —
that pipeline builds and pushes the Docker image; this repo controls what's
actually running in the cluster, driven by Git commits instead of manual
`kubectl` commands.

## 1. Push this repo to GitHub

```bash
cd gitops-argocd-demo
git init
git add .
git commit -m "Initial commit: GitOps manifests for ArgoCD"
git branch -M main
git remote add origin https://github.com/<your-username>/gitops-argocd.git
git push -u origin main
```

## 2. Start your cluster (Minikube or K3s)

**Minikube:**
```bash
minikube start
```

**K3s** (if you're on Linux and prefer a lighter-weight cluster):
```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get nodes
# then set KUBECONFIG to use k3s's kubectl config, e.g.:
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

## 3. Install ArgoCD on the cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all pods to be ready:

```bash
kubectl get pods -n argocd -w
```

(Press Ctrl+C once everything shows `Running`/`Completed`.)

## 4. Access the ArgoCD UI

Port-forward the ArgoCD API server:

```bash
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

Open **https://localhost:8081** in your browser (accept the self-signed cert warning).

Get the initial admin password:

```bash
# PowerShell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | %{[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_))}

# bash/mac/linux
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Log in with:
- Username: `admin`
- Password: (output from the command above)

**Screenshot moment #1:** the ArgoCD login screen / dashboard.

## 5. Connect your Git repo to ArgoCD

Edit `argocd/application.yaml` in this repo and replace `<your-username>`
with your GitHub username, then also update the `image:` line in
`manifests/deployment.yaml` with your Docker Hub username (same as the
CI/CD project). Commit and push:

```bash
git add .
git commit -m "Configure ArgoCD application"
git push
```

## 6. Apply the ArgoCD Application (this is what starts the GitOps sync)

```bash
kubectl apply -f argocd/application.yaml
```

ArgoCD will now:
1. Clone your repo
2. Read the manifests in `manifests/`
3. Apply them to the `default` namespace
4. Continuously watch the repo for changes (auto-sync every ~3 min, or instantly via webhook)

Check status from the CLI:

```bash
kubectl get applications -n argocd
```

Or watch it in the UI — you'll see the `cicd-demo-app` tile go from
**OutOfSync → Syncing → Synced/Healthy**, along with a live resource graph
showing the Deployment, ReplicaSet, Pods, and Service it created.

**Screenshot moment #2:** the Application tile in "Synced/Healthy" state, and
the resource graph view (click into the app).

## 7. Test the GitOps loop: update via Git, not kubectl

This is the actual point of GitOps — change desired state in Git, and the
cluster follows automatically. Try it:

```bash
# e.g. bump replicas or the APP_VERSION env var in manifests/deployment.yaml
```

Edit `manifests/deployment.yaml` — for example change `replicas: 2` to
`replicas: 3`, or bump `APP_VERSION`. Then:

```bash
git add manifests/deployment.yaml
git commit -m "Scale to 3 replicas"
git push
```

Within ArgoCD's sync interval (or click **Refresh** / **Sync** in the UI to
force it immediately), the cluster updates itself:

```bash
kubectl get pods
```

You should see a new pod appear with no `kubectl apply` run by you at all —
ArgoCD did it.

**Screenshot moment #3:** the ArgoCD UI showing a sync event triggered by
your commit (check the app's **History and Rollback** tab, it logs each sync
with the Git commit SHA), plus terminal output of `kubectl get pods` showing
the updated replica count.

## 8. Test self-healing (optional but a great demo of GitOps)

Manually break something on the cluster directly (bypassing Git):

```bash
kubectl scale deployment cicd-demo --replicas=1
```

Because `selfHeal: true` is set in `argocd/application.yaml`, ArgoCD detects
the drift from Git's declared state and reverts it back to what's in the
repo within seconds — without you touching Git or kubectl again.

## 9. Tear down

```bash
kubectl delete -f argocd/application.yaml
kubectl delete namespace argocd
minikube stop
```

## Deliverables checklist

- [ ] Git repo with `manifests/` and `argocd/application.yaml`
- [ ] Screenshot: ArgoCD login/dashboard
- [ ] Screenshot: Application in Synced/Healthy state + resource graph
- [ ] Screenshot: sync history after a Git commit (History and Rollback tab)
- [ ] `NOTES.md` (included) explaining the GitOps flow — use as video script or written notes
