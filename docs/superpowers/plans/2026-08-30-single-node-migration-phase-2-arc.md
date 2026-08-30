# Single-node migration — Phase 2 (ARC cutover) implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the GitHub Actions runner controller and both runner scale sets (`home-ops-runner`, `home-labs-runner`) from the old 3-node cluster to `talos-11`, so `kichi-org/home-ops` and `kichi-org/home-labs` workflows run on the new cluster.

**Architecture:** ARC is the first singleton cutover (spec §8 step 2). A scale set registers with GitHub under its name, so the same name must not be live on two clusters: the `main` PR removes ARC from the old cluster first (Flux prunes it), then the `v2` PR enables the ported copy. Only the work-volume StorageClass, Flux dependencies and the alerts wiring change; the GitHub App credentials (1Password item `actions-runner`) and Talos API access are reused.

**Tech Stack:** Flux, gha-runner-scale-set-controller / gha-runner-scale-set 0.14.2, ESO + 1Password, OpenEBS hostpath, Talos `kubernetesTalosAPIAccess` (`talos.dev/v1alpha1 ServiceAccount`), GitHub REST API.

**Spec:** `docs/superpowers/specs/2026-08-30-single-node-migration-design.md` (§3.5, §8 step 2, §9). Predecessor: `docs/superpowers/plans/2026-08-30-single-node-migration-phase-0-1.md` (execution log holds the live facts).

## Global constraints

- Only one cluster runs ARC at any time (spec §9): old cluster's ARC must be gone from GitHub before the new one is enabled.
- Every change is a PR: `main` for the old cluster, `v2` for the new one; Calvin merges. Commits: subject line only, no body, no trailers.
- New cluster tooling: `cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig`; commit with `mise exec -- git commit`. Old cluster: `cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig`.
- Never re-run `just configure` on `v2`. StorageClass on `v2` is `openebs-hostpath`; the `alerts` Flux component exists on `v2` at `kubernetes/components/alerts`.
- GitHub API calls use the PAT in `~/.claude/settings.json` → `env.GITHUB_PERSONAL_ACCESS_TOKEN` (`gh` is not logged in). Read it with `jq -r .env.GITHUB_PERSONAL_ACCESS_TOKEN ~/.claude/settings.json`; never print it.
- Nothing else on the old cluster changes; old PVCs are untouched (ARC has no persistent data — work volumes are ephemeral).

---

## File map

Branch `v2` (worktree `~/repo/kichi-org/home-ops-v2`):

| Path                                                                                      | Responsibility                                                                                                                                           |
| ----------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `kubernetes/apps/actions-runner-system/namespace.yaml`                                    | namespace (copied from `main`)                                                                                                                           |
| `kubernetes/apps/actions-runner-system/kustomization.yaml`                                | namespace kustomization, `alerts` component                                                                                                              |
| `kubernetes/apps/actions-runner-system/actions-runner-controller/ks.yaml`                 | two Flux Kustomizations: `actions-runner-controller` (depends on `onepassword`), `actions-runner-controller-runners` (depends on controller + `openebs`) |
| `.../actions-runner-controller/app/{kustomization,ocirepository,helmrelease}.yaml`        | controller chart 0.14.2 (copied verbatim)                                                                                                                |
| `.../actions-runner-controller/runners/kustomization.yaml`                                | lists `./home-ops`, `./home-labs`                                                                                                                        |
| `.../runners/home-ops/{externalsecret,helmrelease,kustomization,ocirepository,rbac}.yaml` | `home-ops-runner` scale set; ExternalSecret `home-ops-runner-secret` from item `actions-runner`; storageClass swapped                                    |
| `.../runners/home-labs/{helmrelease,kustomization,rbac}.yaml`                             | `home-labs-runner` scale set (+ NFS `/mnt/data`); storageClass swapped                                                                                   |
| `docs/superpowers/plans/2026-08-30-single-node-migration-phase-2-arc.md`                  | this plan (copied so it survives `v2 → main`)                                                                                                            |

Branch `main` (`~/repo/kichi-org/home-ops`): delete `kubernetes/apps/actions-runner-system/` (whole directory). Flux `prune: true` on `cluster-apps` removes the namespace's Kustomizations, and their own `prune: true` removes the HelmReleases, so GitHub deregisters the scale sets.

---

### Task 1: Port ARC to `v2` on a feature branch and open the PR (not merged yet)

**Files:** see file map (all under `kubernetes/apps/actions-runner-system/` on `v2`).

**Produces:** PR "feat(actions-runner-controller): port runners to the new cluster" against `v2`, validated by kustomize and flate CI, **left open**.

- [ ] **Step 1: Branch and copy**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH
git checkout -b feat/arc v2
cp -R ../home-ops/kubernetes/apps/actions-runner-system kubernetes/apps/actions-runner-system
cp ../home-ops/docs/superpowers/plans/2026-08-30-single-node-migration-phase-2-arc.md docs/superpowers/plans/
find kubernetes/apps/actions-runner-system -type f | wc -l
```

Expected: `15` files.

- [ ] **Step 2: Swap the StorageClass and the Flux dependencies**

```bash
cd ~/repo/kichi-org/home-ops-v2
sed -i '' 's/storageClassName: longhorn/storageClassName: openebs-hostpath/' kubernetes/apps/actions-runner-system/actions-runner-controller/runners/home-ops/helmrelease.yaml kubernetes/apps/actions-runner-system/actions-runner-controller/runners/home-labs/helmrelease.yaml
sed -i '' 's/fsGroup: 121 # required for longhorn pvc behavior/fsGroup: 121/' kubernetes/apps/actions-runner-system/actions-runner-controller/runners/home-ops/helmrelease.yaml kubernetes/apps/actions-runner-system/actions-runner-controller/runners/home-labs/helmrelease.yaml
python3 - <<'EOF'
p='kubernetes/apps/actions-runner-system/actions-runner-controller/ks.yaml'; s=open(p).read()
old='''  name: actions-runner-controller
spec:
  interval: 1h
'''
new='''  name: actions-runner-controller
spec:
  dependsOn:
    - name: onepassword
      namespace: external-secrets
  interval: 1h
'''
assert old in s; s=s.replace(old,new)
old='''  dependsOn:
    - name: actions-runner-controller
    - name: longhorn
      namespace: longhorn-system
'''
new='''  dependsOn:
    - name: actions-runner-controller
    - name: openebs
      namespace: openebs-system
'''
assert old in s; s=s.replace(old,new); open(p,'w').write(s)
EOF
grep -rn 'longhorn' kubernetes/apps/actions-runner-system || echo "no longhorn refs"
grep -n -A3 'dependsOn' kubernetes/apps/actions-runner-system/actions-runner-controller/ks.yaml
```

Expected: `no longhorn refs`; the controller depends on `onepassword`, the runners on `actions-runner-controller` + `openebs`. The namespace `kustomization.yaml` already carries `components: [../../components/alerts]` — keep it.

- [ ] **Step 3: Validate the build and the chart values**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH
for d in actions-runner-system actions-runner-system/actions-runner-controller/app actions-runner-system/actions-runner-controller/runners; do kubectl kustomize kubernetes/apps/$d >/dev/null && echo "OK $d"; done
kubectl kustomize kubernetes/apps/actions-runner-system/actions-runner-controller/runners | grep -E 'kind:|storageClassName|githubConfigUrl|name: home-' | sort | uniq -c
```

Expected: three `OK`; two HelmReleases, two `storageClassName: openebs-hostpath`, `githubConfigUrl` for both repos, RBAC objects for `home-ops-runner` and `home-labs-runner`.

- [ ] **Step 4: Confirm the new cluster has what the runners need (pre-flight, read-only)**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl get crd serviceaccounts.talos.dev
kubectl get clustersecretstore onepassword --no-headers
op item get actions-runner --vault Kubernetes --format json | jq -r '.fields[] | select(.label|test("ACTIONS_RUNNER")) | .label'
kubectl run nfsprobe --rm -i --restart=Never --image=busybox:1.36 --overrides='{"spec":{"containers":[{"name":"n","image":"busybox:1.36","command":["sh","-c","ls /mnt/data | head -3"],"volumeMounts":[{"name":"d","mountPath":"/mnt/data"}]}],"volumes":[{"name":"d","nfs":{"server":"kl-san-1.localdomain","path":"/volume1/data"}}]}}' 2>&1 | grep -v '^pod '
```

Expected: the Talos `ServiceAccount` CRD exists (from `kubernetesTalosAPIAccess`), store `Valid`, the three `ACTIONS_RUNNER_*` fields are present, and the NFS probe lists directories (proves `kl-san-1.localdomain` resolves and the export accepts `.111`).

- [ ] **Step 5: Commit, push, open the PR against `v2`**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH
git add kubernetes/apps/actions-runner-system docs && mise exec -- git commit -q -m "feat(actions-runner-controller): port runners to the new cluster" && git push -u origin feat/arc
```

Then GitHub MCP `create_pull_request` (owner `kichi-org`, repo `home-ops`, head `feat/arc`, **base `v2`**), body: "Phase 2 of the single-node migration — enables ARC on talos-11. Merge only after the `main` removal PR has pruned the old scale sets (see plan Task 2)." Record the PR number as `PR_V2`.
Expected: PR open, `flate` check green. **Do not merge yet.**

### Task 2: Remove ARC from the old cluster (`main` PR, merged first)

**Files:** Delete `kubernetes/apps/actions-runner-system/` on `main`.

**Produces:** old cluster without ARC; GitHub shows zero runners for both repos.

- [ ] **Step 1: Snapshot the old state for the rollback note**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
kubectl -n actions-runner-system get helmrelease,pods --no-headers | awk '{print $1,$2,$3}'
TOKEN=$(jq -r .env.GITHUB_PERSONAL_ACCESS_TOKEN ~/.claude/settings.json)
for r in home-ops home-labs; do curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/kichi-org/$r/actions/runners" | jq -r --arg r "$r" '"\($r): \(.total_count) runner(s) \([.runners[].name] | join(","))"'; done
```

Expected: two HelmReleases, controller + two listener pods; runner counts ≥ 0 (scale sets show as runners only while a job runs, listeners always exist).

- [ ] **Step 2: Branch, delete, PR**

```bash
cd ~/repo/kichi-org/home-ops
git stash push -q -- .gitignore 2>/dev/null || true
git checkout -b chore/arc-cutover main
git rm -r -q kubernetes/apps/actions-runner-system
git commit -q -m "chore(actions-runner-controller): move runners to the new cluster"
git push -u origin chore/arc-cutover
git checkout -q main && git stash pop -q 2>/dev/null || true
```

GitHub MCP `create_pull_request` (head `chore/arc-cutover`, base `main`), body: "Phase 2 of the single-node migration — removes ARC from the old cluster; the `v2` PR (#PR_V2) enables it on talos-11 once this is pruned." Record as `PR_MAIN`.
Expected: `flux-local` diff comment shows only deletions under `actions-runner-system`.

- [ ] **Step 3: Calvin merges `PR_MAIN`; watch the prune**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
flux --namespace flux-system reconcile kustomization flux-system --with-source | tail -1
until ! kubectl get kustomization -n actions-runner-system --no-headers 2>/dev/null | grep -q .; do sleep 10; done; echo "kustomizations pruned"
until ! kubectl -n actions-runner-system get pods --no-headers 2>/dev/null | grep -q .; do sleep 10; done; echo "pods gone"
TOKEN=$(jq -r .env.GITHUB_PERSONAL_ACCESS_TOKEN ~/.claude/settings.json)
for r in home-ops home-labs; do curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/kichi-org/$r/actions/runners" | jq -r --arg r "$r" '"\($r): \(.total_count) runner(s)"'; done
```

Expected: Kustomizations and pods gone within a few minutes; both repos report `0 runner(s)`. If a listener lingers because the HelmRelease uninstall hangs on the `AutoscalingRunnerSet` finalizer, `kubectl -n actions-runner-system delete autoscalingrunnerset --all` and re-check. The namespace object itself may remain (namespace.yaml prune) — harmless.

### Task 3: Enable ARC on the new cluster (merge `PR_V2`)

**Files:** none new; merge only.

- [ ] **Step 1: Calvin merges `PR_V2`; reconcile and wait**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
git checkout -q v2 && git pull -q --ff-only && git branch -d feat/arc
flux --namespace flux-system reconcile kustomization flux-system --with-source | tail -1
until kubectl get kustomization -n actions-runner-system actions-runner-controller-runners -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null | grep -q True; do sleep 15; done
kubectl -n actions-runner-system get helmrelease,pods --no-headers | awk '{print $1,$2,$3}'
kubectl -n actions-runner-system get secret home-ops-runner-secret -o jsonpath='{.data}' | jq 'keys'
kubectl -n actions-runner-system get secret home-ops-runner home-labs-runner -o jsonpath='{range .items[*]}{.metadata.name}: {.type}{"\n"}{end}'
```

Expected: HelmReleases `actions-runner-controller`, `home-ops-runner`, `home-labs-runner` Ready; pods `actions-runner-controller-*`, `home-ops-runner-*-listener`, `home-labs-runner-*-listener` Running; secret keys `github_app_id`, `github_app_installation_id`, `github_app_private_key`; the two Talos-issued secrets exist (type `Opaque`, created by Talos from the `talos.dev` ServiceAccounts).

- [ ] **Step 2: Confirm GitHub sees the scale sets from the new cluster**

```bash
TOKEN=$(jq -r .env.GITHUB_PERSONAL_ACCESS_TOKEN ~/.claude/settings.json)
for r in home-ops home-labs; do curl -s -H "Authorization: Bearer $TOKEN" "https://api.github.com/repos/kichi-org/$r/actions/runners" | jq -r --arg r "$r" '"\($r): \(.total_count) runner(s) \([.runners[].name] | join(","))"'; done
kubectl -n actions-runner-system logs deploy/actions-runner-controller --since=10m | grep -ciE 'error' || true
```

Expected: listener registration visible in the controller logs without errors (runner count stays 0 until a job runs — that's normal for `minRunners: 0`).

### Task 4: Smoke-test both runner sets with real workflows

**Files:** none.

- [ ] **Step 1: Dispatch `Test` on home-ops (`runs-on: home-ops-runner`, branch `main`)**

```bash
TOKEN=$(jq -r .env.GITHUB_PERSONAL_ACCESS_TOKEN ~/.claude/settings.json)
curl -s -o /dev/null -w 'dispatch=%{http_code}\n' -X POST -H "Authorization: Bearer $TOKEN" -H 'Accept: application/vnd.github+json' https://api.github.com/repos/kichi-org/home-ops/actions/workflows/test.yaml/dispatches -d '{"ref":"main"}'
sleep 20
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl -n actions-runner-system get pods --no-headers | grep -E 'home-ops-runner-[a-z0-9]+-runner' | awk '{print $1,$3}'
until [ "$(curl -s -H "Authorization: Bearer $TOKEN" 'https://api.github.com/repos/kichi-org/home-ops/actions/workflows/test.yaml/runs?per_page=1' | jq -r '.workflow_runs[0].status')" = "completed" ]; do sleep 15; done
curl -s -H "Authorization: Bearer $TOKEN" 'https://api.github.com/repos/kichi-org/home-ops/actions/workflows/test.yaml/runs?per_page=1' | jq -r '.workflow_runs[0] | "\(.conclusion) \(.html_url)"'
```

Expected: `dispatch=204`; an ephemeral `home-ops-runner-…-runner` pod appears on talos-11 (with a hostpath work PVC) and the run ends `success`.

- [ ] **Step 2: Dispatch `tvb-postprocess` on home-labs in dry-run (`runs-on: home-labs-runner`, exercises the NFS mount)**

```bash
TOKEN=$(jq -r .env.GITHUB_PERSONAL_ACCESS_TOKEN ~/.claude/settings.json)
curl -s -o /dev/null -w 'dispatch=%{http_code}\n' -X POST -H "Authorization: Bearer $TOKEN" -H 'Accept: application/vnd.github+json' https://api.github.com/repos/kichi-org/home-labs/actions/workflows/tvb-postprocess.yaml/dispatches -d '{"ref":"main","inputs":{"dry_run":"true"}}'
until [ "$(curl -s -H "Authorization: Bearer $TOKEN" 'https://api.github.com/repos/kichi-org/home-labs/actions/workflows/tvb-postprocess.yaml/runs?per_page=1' | jq -r '.workflow_runs[0].status')" = "completed" ]; do sleep 15; done
curl -s -H "Authorization: Bearer $TOKEN" 'https://api.github.com/repos/kichi-org/home-labs/actions/workflows/tvb-postprocess.yaml/runs?per_page=1' | jq -r '.workflow_runs[0] | "\(.conclusion) \(.html_url)"'
```

Expected: `dispatch=204`, run `success` (dry-run touches nothing on the NAS). If home-labs' default branch is not `main`, use `GET /repos/kichi-org/home-labs` → `.default_branch` for `ref`.

- [ ] **Step 3: Confirm the ephemeral work PVCs are cleaned up**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
sleep 60; kubectl -n actions-runner-system get pvc,pods --no-headers | awk '{print $1,$2,$3}'
```

Expected: only the controller and the two listener pods; no leftover `*-work` PVCs (ARC deletes them with the runner pod).

### Task 5: Close out

**Files:** `docs/superpowers/plans/2026-08-30-single-node-migration-phase-2-arc.md` (execution log, both branches); memory `migration-progress`.

- [ ] **Step 1: Record the execution log** — fill in the section below (PR numbers, run URLs, date) on `v2` (`docs/…`) and on `main`; commit each as `docs(plan): phase 2 execution log` and push (`main` is docs-only, push directly; `v2` likewise).

- [ ] **Step 2: Update memory** — `migration-progress`: ARC cut over on the date, spec §8 step 2 done, next = step 3 (TeslaMate + Grafana).

- [ ] **Step 3: Rollback note (only if something regressed)** — revert `PR_V2` on `v2` first (prunes the new scale sets), wait for GitHub to show `0 runner(s)`, then revert `PR_MAIN` on `main`; the old cluster re-registers the scale sets within a few minutes.

---

## Execution log

- PR_V2 / PR_MAIN: _(Task 1 / Task 2)_
- Old-cluster prune confirmed: _(Task 2 Step 3)_
- Smoke runs: home-ops `Test` _(url)_, home-labs `tvb-postprocess` dry-run _(url)_
- Date completed: _(Task 5)_
