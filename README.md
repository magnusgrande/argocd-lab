# argocd-lab

Personal sandbox for learning Argo CD. Standalone repo — separate from the Kystverket project.

## Layout

```
bootstrap/   applied by hand once: Helm values for Argo CD + the root Application
apps/        what the root app syncs. A file here = an Application in the cluster
extras/      disabled apps for later exercises. Enable one by moving it into apps/
workloads/   the manifests the Applications point at
src/web/     tiny app built by CI (exercise 10)
mise-tasks/  up / down / password / login / set-repo
```

Everything after `mise run up` happens through Git: to change the cluster, you commit.

## Setup

1. Create a GitHub repo `argocd-lab` on your own account. **Public** is easiest (nothing secret here).
   Private also works: put a fine-grained PAT (this repo only, Contents: read) in `mise.local.toml`:
   ```toml
   [env]
   GITHUB_TOKEN = "github_pat_..."
   ```
   Different repo URL? Set `REPO_URL` in `mise.toml`, then `mise run set-repo`.
2. Init and push:
   ```bash
   git init -b main && git add -A
   git update-index --chmod=+x mise-tasks/*
   git commit -m "Initial lab"
   git remote add origin https://github.com/magnusgrande/argocd-lab.git
   git push -u origin main
   ```
3. `mise install && mise run up` (Podman machine must be rootful, same as the project's kind cluster).
4. Open http://localhost:18080, user `admin`, password from `mise run password`. Optionally `mise run login` for the CLI.

Git is polled every 60s (`timeout.reconciliation`). Press **Refresh** in the UI to skip the wait.
Webhooks can't reach localhost, so polling is the only trigger here.

Tear down: `mise run down`.

## Exercises

Each one: **do** → **notice**. Write findings down as you go (bottom of this file).

### 1. Manual sync
**Do:** In the UI, open `root` → it created the `hello` app. `hello` is *OutOfSync*. Click it, look at **Diff**, then **Sync**. Open http://localhost:18081.
**Notice:** Argo CD's two questions: *Sync status* (Git vs. cluster) and *Health* (is it actually working). They're independent.

### 2. Drift
**Do:** `kubectl -n lab-hello scale deploy hello --replicas 3`.
**Notice:** App goes *OutOfSync*, diff shows `replicas`. Nothing reverts it — yet. Sync to fix.

### 3. Git is the source of truth
**Do:** In `apps/hello.yaml`, add under `syncPolicy`:
```yaml
    automated:
      prune: true
      selfHeal: true
```
Push. Repeat exercise 2. Then change `PODINFO_UI_MESSAGE` in `workloads/hello/deployment.yaml` and push.
**Notice:** Manual changes are reverted in seconds. Git changes roll out without you running anything. Also note you changed an *Application* through Git — the root app synced it.

### 4. Prune and history
**Do:** Delete `workloads/hello/service.yaml`, push. Then `git revert HEAD`, push. Look at **History and Rollback** in the app.
**Notice:** `prune: true` deletes what's gone from Git. UI rollback exists, but with auto-sync on it's disabled — the real rollback is `git revert`.

### 5. Helm chart as a source
**Do:** `git mv extras/podinfo-helm.yaml apps/`, push. Open http://localhost:18083. Run `helm list -A`.
**Notice:** No Helm release exists — Argo CD only uses Helm to *render*. Hooks/`lookup`/release history behave differently than `helm install`. Compare with `helm_release`/Helmfile in the project.

### 6. ApplicationSet (one app per environment)
**Do:** `git mv extras/envs-appset.yaml apps/`, push → `envs-dev`, `envs-prod`. Then `cp -r workloads/envs/overlays/dev workloads/envs/overlays/staging`, edit the message, push. Delete it again.
**Notice:** Apps are *generated* from folder structure. Other generators: list, cluster (same app to many clusters), pull request (preview env per PR), matrix.

### 7. Sync hooks and waves
**Do:** `git mv extras/hooks/*.yaml workloads/hello/`, push, watch the sync. Then make the PreSync job `exit 1`.
**Notice:** Ordering: PreSync → resources (by `argocd.argoproj.io/sync-wave`, low first) → PostSync. A failed PreSync blocks the deploy — this is where DB migrations go. Try adding `argocd.argoproj.io/sync-wave: "-1"` to the Service.

### 8. Projects and scoped credentials
**Do:** `git mv extras/project-lab.yaml apps/`, push. Change `project: default` → `project: lab` in `apps/hello.yaml`, push. Then try `namespace: kube-system` as destination.
Then: `argocd proj role create-token lab deployer` and use that token with `argocd app sync hello --auth-token ...` — then try syncing `root` with it.
**Notice:** AppProject = allowed repos, destinations, and resource kinds. The role token can sync lab apps and nothing else. Compare with the ARC runner permission split.

### 9. Argo CD manages itself
**Do:** `git mv extras/argocd-self.yaml apps/`, push. Then change `timeout.reconciliation` in `bootstrap/argocd-values.yaml`, push.
**Notice:** Argo CD is now upgraded/configured by commits (`mise run up` on an existing cluster would now fight it — only use it on a fresh one). The bootstrap problem remains: *something* has to install it the first time (here `mise run up`; at Kystverket, probably Terraform).

### 10. CI/CD loop
**Do:**
1. Actions → run *web - build and bump* manually. It pushes an image to ghcr.io and commits the tag.
2. GitHub → your profile → Packages → `argocd-lab-web` → make it **public** (or add an imagePullSecret).
3. `git pull`, `git mv extras/web.yaml apps/`, push. Open http://localhost:18082.
4. Edit `src/web/index.html`, push. Watch: Actions build → tag commit → Argo CD rollout.

**Notice:** CI never talks to the cluster. It needs registry + repo write, **no kubeconfig**. The cluster pulls from GitHub; nothing has to reach *into* it. That's the pull model, and it sidesteps the "GitHub runners can't reach the cluster" problem that led to ARC in the project.

### Further, if you want
- **Kargo** (Kystverket is looking at it): promotion between stages (dev → prod) on top of Argo CD, replacing hand-rolled tag bumps.
- **Argo CD Image Updater**: Argo CD writes the tag back itself, CI only pushes the image.
- **Secrets**: nothing secret belongs in Git. Look at Sealed Secrets or External Secrets Operator.
- **Forgejo as the source**: point an app at an in-cluster Forgejo repo — answers whether this works fully offline/on-prem.
- **Notifications**: `notifications.enabled: true` → Slack/Teams on sync failure.

## Findings

Questions worth answering for the project:

- Where's the line between Terraform and Argo CD? (Cluster/bootstrap vs. in-cluster workloads?)
- What would `bootstrap-all` look like as Terraform (install Argo CD + root app) + Argo CD (everything else)?
- Does pull-based deploy fit Kystverket's firewall/runner constraints better than push-based?
- Cost: another always-on component with cluster-admin to operate, upgrade, and secure.

-
