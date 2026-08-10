# GitOps Flow — Explanation Notes

## What GitOps is

GitOps is a way of managing infrastructure and deployments where **Git is
the single source of truth** for what should be running in a cluster. Instead
of running `kubectl apply` by hand, you commit the desired state (YAML
manifests) to a Git repo, and an agent running inside the cluster
(ArgoCD, in this case) continuously compares the live cluster state against
what's declared in Git — and reconciles any difference automatically.

## The flow, step by step

1. **Desired state lives in Git.** The `manifests/` folder in this repo
   defines exactly what should be running: which image, how many replicas,
   what ports, etc. This is declarative — it describes the end state, not
   the steps to get there.

2. **ArgoCD watches the repo.** After the `Application` custom resource
   (`argocd/application.yaml`) is applied, ArgoCD polls this Git repo/path on
   an interval (default ~3 minutes, or instantly via a webhook) and compares
   it to what's actually deployed in the cluster.

3. **Diff detection.** If the repo state and the cluster state don't match,
   ArgoCD marks the Application `OutOfSync`.

4. **Auto-sync.** Because `syncPolicy.automated` is enabled, ArgoCD
   automatically applies the difference — creating, updating, or deleting
   Kubernetes resources so the cluster matches Git. No human runs `kubectl
   apply`.

5. **Self-healing.** If someone changes something directly on the cluster
   (e.g., `kubectl scale` or `kubectl edit`) without going through Git,
   ArgoCD detects that drift and reverts it back to match Git's declared
   state, because `selfHeal: true` is set. Git remains authoritative even
   against manual cluster changes.

6. **Audit trail for free.** Because every change to the cluster's desired
   state was a Git commit, you get a full history of *who* changed *what*
   and *when* — visible both in `git log` and in ArgoCD's own sync history,
   which ties each sync back to the exact commit SHA that triggered it.

7. **Rollback is a Git revert.** To roll back a bad deploy, you don't run
   imperative kubectl commands — you `git revert` the commit, push, and
   ArgoCD syncs the cluster back to the previous state automatically.

## Why this matters vs. plain `kubectl apply`

| Manual `kubectl apply`                        | GitOps with ArgoCD                                  |
|------------------------------------------------|-------------------------------------------------------|
| No single source of truth — cluster state and repo can silently drift | Git is the source of truth; drift is auto-corrected |
| Rollback means remembering/finding the old YAML | Rollback is `git revert`                             |
| No built-in audit trail of who changed what     | Every change is a Git commit with author + timestamp |
| Deploy access requires cluster credentials      | Deploy access is just Git write access               |
| Easy for cluster state to become undocumented   | Cluster state is always exactly what's in the repo   |

## How this maps to what was actually done in this project

- `manifests/deployment.yaml` and `manifests/service.yaml` — the declarative
  desired state, version-controlled in this repo.
- `argocd/application.yaml` — tells ArgoCD which repo, which path, which
  cluster/namespace, and turns on `automated` sync with `selfHeal: true`.
- Editing `replicas` (or any field) in `manifests/deployment.yaml`, then
  `git commit` + `git push`, is the entire "deploy" action — ArgoCD does the
  rest.
- Running `kubectl scale` directly against the cluster and watching ArgoCD
  revert it demonstrates self-healing — the cluster can't drift from Git for
  long even if someone bypasses the GitOps process.

## Suggested narration if recording a video walkthrough

1. Show the ArgoCD dashboard — app is Synced/Healthy.
2. Open `manifests/deployment.yaml` on GitHub, show current replica count.
3. Edit it locally, change replicas, commit, push.
4. Switch to ArgoCD UI, hit Refresh (or wait for auto-sync), show status
   flip to Syncing then back to Synced, and the new pod appearing.
5. Run `kubectl get pods` in a terminal to confirm the extra pod exists.
6. Optionally: run `kubectl scale` manually to break it, show ArgoCD
   self-heal it back within seconds — this is the strongest visual proof
   that Git, not the cluster, is authoritative.
