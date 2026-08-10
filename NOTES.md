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

## Video script (read-aloud, ~3–4 min)

This is written to match a screen recording with roughly one cut per
paragraph. Swap in your own repo URL/username where noted.

---

**[Scene 1 — ArgoCD dashboard, app tile visible]**

"This is ArgoCD running on a local Minikube cluster. It's watching a GitHub
repo called `gitops-argocd`, and right now the application `cicd-demo-app`
shows Healthy and Synced — meaning everything running in the cluster
matches exactly what's declared in that repo's `manifests` folder."

**[Scene 2 — click into the app, show the resource graph]**

"Here's the resource graph. ArgoCD created a Deployment, which created a
ReplicaSet, which created the actual Pods, plus a Service to expose them —
all from a single YAML file I never manually applied with kubectl. I only
applied one thing myself: the ArgoCD `Application` resource that tells it
which repo and path to watch."

**[Scene 3 — terminal, run `kubectl get pods`]**

"From the cluster side, `kubectl get pods` confirms two pods running —
matching `replicas: 2` in the deployment manifest in Git."

**[Scene 4 — GitHub, open `manifests/deployment.yaml`, edit `replicas`]**

"Now here's the actual GitOps loop. Instead of running a kubectl command to
scale this, I'm going to change the desired state in Git. I'll edit
`replicas: 2` to `replicas: 3` directly in the manifest, commit, and push."

**[Scene 5 — terminal, `git add` / `git commit` / `git push`]**

"That's it — that's the entire 'deployment' action from my side. No cluster
credentials needed, just Git write access."

**[Scene 6 — ArgoCD UI, hit Refresh, status flips]**

"Back in ArgoCD, I'll hit Refresh so it doesn't wait for its normal polling
interval. Watch the status — it goes from Synced to OutOfSync, because it
detected the repo no longer matches the cluster. Then Syncing, as it
reconciles the difference. And back to Synced — a third pod has appeared."

**[Scene 7 — terminal, `kubectl get pods` again]**

"Confirmed from the cluster side too — three pods now, and I never touched
kubectl to make that scaling change happen. Git was the only thing I
edited."

**[Scene 8 — terminal, `kubectl scale deployment cicd-demo --replicas=1`]**

"Last part — self-healing. I'm going to bypass Git entirely and scale the
deployment down to one pod directly on the cluster, the old-fashioned way."

**[Scene 9 — terminal, `kubectl get pods`, pause, run again]**

"For a moment it drops to one pod. But because the ArgoCD Application has
`selfHeal: true` set, it detects that the cluster no longer matches Git's
declared state of three replicas — and reverts my manual change
automatically, within seconds. I didn't run any command to fix it; ArgoCD
did."

**[Scene 10 — ArgoCD UI, show it briefly flashed OutOfSync then back to Synced]**

"And here in the UI you can see that exact moment — it briefly flagged
OutOfSync from my manual scale command, then self-corrected back to Synced.
That's the core idea of GitOps: Git is the only authoritative source of
truth, and the cluster can't drift from it for long, even if someone
bypasses the process and touches it directly."

---

## Quick reference — what actually happened in this session

- ArgoCD installed on Minikube via the standard install manifest
- Application `cicd-demo-app` pointed at a GitHub repo's `manifests/` folder
- Auto-sync + self-heal enabled from the start (`syncPolicy.automated`)
- Scaling tested two ways: via a Git commit (the intended path) and via a
  direct `kubectl scale` (to prove drift correction works)
- Both screenshots and this script can be paired for a submission video, or
  the script alone can stand in as written notes if no video is required.