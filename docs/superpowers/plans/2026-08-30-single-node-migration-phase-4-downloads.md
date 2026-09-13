# Single-node migration — Phase 4 (downloads: *arr + qBittorrent + qui + Recyclarr) implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move the `downloads` namespace (Sonarr, Radarr, Prowlarr, qBittorrent, qui, Recyclarr) to `talos-11` with their configuration intact, qBittorrent back on `172.16.0.123` so the existing UniFi port-forward keeps working, and every config PVC backed up hourly by VolSync from day one.

**Architecture:** Config data is small (~180 MB total) and lives in Helm-managed Longhorn PVCs that **are deleted when the old HelmReleases are uninstalled**, so the copy happens _before_ the `main` PR is merged: old apps are scaled to 0 (HelmReleases suspended so Flux does not scale them back), each `/config` is tarred to an NFS staging directory (`kl-san-1:/volume1/data/_migration`), then the `main` PR prunes the old apps. The `v2` PR (merged second) brings the apps up on fresh hostpath PVCs, which are then overwritten from the staging tars with the apps scaled to 0. qBittorrent's `.123` is only enabled on `v2` after the old one is gone (L2 announcement must be unique) — the `v2` pool gets a second block for it.

**Tech Stack:** Flux, bjw-s app-template 5.1.0, home-operations images, ESO + 1Password (items `sonarr`, `radarr`, `prowlarr`, `qui`, `recyclarr`), OpenEBS hostpath, VolSync component (`kubernetes/components/volsync`), Cilium LB-IPAM, NFS.

**Spec:** `docs/superpowers/specs/2026-08-30-single-node-migration-design.md` (§3.3, §3.5, §6.2 layer 2, §8 step 4, §9). Predecessors: phase 0–1, 2, 3 plans in this directory.

## Global constraints

- `.123` is announced by exactly one cluster at a time (spec §9).
- The old media on NFS (`/volume1/data`) is shared and untouched; only `/volume1/data/_migration/` is created (and removed at the end).
- Every change is a PR: `main` for the old cluster, `v2` for the new; Calvin merges. Commits: subject line only, no body, no trailers.
- New cluster tooling: `cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig`; commit with `mise exec -- git commit`. Old cluster: `cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig`.
- Lesson from phase 3: anything needed from the old cluster (secrets, data) is captured **before** the `main` PR is merged; ESO-owned Secrets and Helm-owned PVCs vanish with the prune.
- All six apps run as uid/gid 1000 (`fsGroup: 1000`), matching the VolSync component defaults.

---

## File map

Branch `v2` (worktree `~/repo/kichi-org/home-ops-v2`):

| Path                                                                           | Responsibility                                                                |
| ------------------------------------------------------------------------------ | ----------------------------------------------------------------------------- |
| `kubernetes/apps/downloads/{namespace,kustomization}.yaml`                     | namespace + `alerts` component, six `ks.yaml` entries (copied)                |
| `kubernetes/apps/downloads/<app>/ks.yaml` (×6)                                 | deps → `openebs`, `onepassword`; VolSync component for all except `recyclarr` |
| `kubernetes/apps/downloads/<app>/app/*` (×6)                                   | copied; `storageClass: openebs-hostpath`, `retain: true` on the config PVC    |
| `kubernetes/apps/kube-system/cilium/app/networks.yaml`                         | pool gains block `172.16.0.123–172.16.0.123`                                  |
| `docs/superpowers/plans/2026-08-30-single-node-migration-phase-4-downloads.md` | this plan                                                                     |

Branch `main`: delete `kubernetes/apps/downloads/` (whole namespace directory).

Scratch (not in git): `/volume1/data/_migration/<app>.tar` on the NAS, created by helper pods.

---

### Task 1: Port the namespace to `v2` on a feature branch and open the PR (not merged yet)

**Files:** see file map.

**Produces:** PR "feat(downloads): port *arr, qbittorrent, qui, recyclarr to the new cluster" against `v2`, left open. Record as `PR_V2`.

- [ ] **Step 1: Branch, copy, rewrite storage/deps/VolSync, extend the pool**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH
git checkout -b feat/downloads v2
cp -R ../home-ops/kubernetes/apps/downloads kubernetes/apps/downloads
cp ../home-ops/docs/superpowers/plans/2026-08-30-single-node-migration-phase-4-downloads.md docs/superpowers/plans/
python3 - <<'EOF'
import re, pathlib
root = pathlib.Path('kubernetes/apps/downloads')
for app in ['prowlarr','qbittorrent','qui','radarr','recyclarr','sonarr']:
    # ks.yaml: deps + volsync component
    p = root/app/'ks.yaml'; s = p.read_text()
    old = '  dependsOn:\n    - name: longhorn\n      namespace: longhorn-system\n'
    assert old in s, app
    new = '  dependsOn:\n    - name: openebs\n      namespace: openebs-system\n    - name: onepassword\n      namespace: external-secrets\n'
    if app != 'recyclarr':
        new = '  components:\n    - ../../../../components/volsync\n' + new
    s = s.replace(old, new)
    if app == 'prowlarr':
        s = s.replace('      VOLSYNC_CAPACITY: 1Gi\n', '')
    p.write_text(s)
    # helmrelease: storage class + retain
    p = root/app/'app'/'helmrelease.yaml'; s = p.read_text()
    old = '        type: persistentVolumeClaim\n        storageClass: longhorn\n'
    assert old in s, app
    s = s.replace(old, '        type: persistentVolumeClaim\n        storageClass: openebs-hostpath\n        retain: true\n')
    s = s.replace('        # existingClaim: "{{ .Release.Name }}"\n', '')
    p.write_text(s)
# cilium pool: add .123 block
p = pathlib.Path('kubernetes/apps/kube-system/cilium/app/networks.yaml'); s = p.read_text()
old = '    - start: "172.16.0.30"\n      stop: "172.16.0.99"\n'
assert old in s
p.write_text(s.replace(old, old + '    - start: "172.16.0.123"\n      stop: "172.16.0.123"\n'))
EOF
grep -rn 'longhorn' kubernetes/apps/downloads || echo "no longhorn refs"
grep -c 'retain: true' kubernetes/apps/downloads/*/app/helmrelease.yaml
```

Expected: `no longhorn refs`; each of the six helmreleases reports `1`.

- [ ] **Step 2: Validate builds and that app-template accepts `retain`**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH
kubectl kustomize kubernetes/apps/downloads >/dev/null && echo "ns OK"
for a in prowlarr qbittorrent qui radarr recyclarr sonarr; do kubectl kustomize kubernetes/apps/downloads/$a/app >/dev/null && echo "OK $a"; done
helm template sonarr oci://ghcr.io/bjw-s-labs/helm/app-template --version 5.1.0 -n downloads -f <(yq '.spec.values' kubernetes/apps/downloads/sonarr/app/helmrelease.yaml) | grep -n -B2 -A2 'resource-policy' | head
kubectl kustomize kubernetes/apps/kube-system/cilium/app | grep -A6 'blocks:'
```

Expected: all OK; the rendered PVC carries `helm.sh/resource-policy: keep` (if `helm template` rejects `retain`, remove that line from the six helmreleases — the PVCs then stay Helm-owned, which is what `main` has today); pool shows both blocks.

- [ ] **Step 3: Pre-flight on the new cluster (read-only)**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
for i in sonarr radarr prowlarr qui recyclarr; do op item get $i --vault Kubernetes --format json | jq -r --arg i "$i" '"\($i): " + ([.fields[] | select(.label!="notesPlain" and .label!="password" and .label!="username") | .label] | join(","))'; done
kubectl get clustersecretstore onepassword --no-headers | awk '{print $1,$3,$5}'
```

Expected: each item lists its `*_API_KEY` (qui: whatever `qui/app/externalsecret.yaml` extracts); store Valid. If `op` times out on its auth prompt, skip — the ExternalSecrets will surface a missing field as `SecretSyncedError` in Task 4.

- [ ] **Step 4: Commit, push, open the PR**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH
git add kubernetes/apps/downloads kubernetes/apps/kube-system/cilium docs && mise exec -- git commit -q -m "feat(downloads): port arr stack, qbittorrent, qui and recyclarr" && git push -u origin feat/downloads
```

GitHub MCP `create_pull_request` (head `feat/downloads`, base `v2`), body: "Phase 4 — enables the downloads namespace on talos-11 (hostpath PVCs with retain, VolSync on all config PVCs, qBittorrent pinned to .123 via a new pool block). Merge only after the `main` removal PR has pruned the old apps (plan Task 3)." **Do not merge yet.**

### Task 2: Freeze the old apps and stage their configs on the NAS

**Files:** none in git. Output: `/volume1/data/_migration/{sonarr,radarr,prowlarr,qbittorrent,qui}.tar`.

**Interfaces:**

- Consumes: old cluster PVCs `downloads/<app>` (Longhorn RWO), NFS `kl-san-1.localdomain:/volume1/data`.
- Produces: one tar per app containing the contents of `/config` (relative paths), owned by uid 1000.

- [ ] **Step 1: Suspend the HelmReleases and scale the apps to 0 (qBittorrent last — active torrents pause)**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
for a in sonarr radarr prowlarr qui qbittorrent; do flux -n downloads suspend helmrelease $a; done
for a in sonarr radarr prowlarr qui qbittorrent; do kubectl -n downloads scale deploy/$a --replicas=0; done
until ! kubectl -n downloads get pods --no-headers | grep -vE 'recyclarr' | grep -q .; do sleep 5; done; echo "old downloads apps stopped $(date +%T)"
kubectl -n downloads get pvc --no-headers | awk '{print $1,$2}'
```

Expected: five `suspended`, five `scaled`, only the completed `recyclarr-*` job pod remains; PVCs still Bound. Record the timestamp — start of the downloads outage.

- [ ] **Step 2: Tar each config PVC to NFS with a helper pod (one pod per app; Longhorn RWO attaches to the helper now that the app is gone)**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
for a in sonarr radarr prowlarr qui qbittorrent; do
kubectl -n downloads apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: {name: stage-$a}
spec:
  restartPolicy: Never
  securityContext: {runAsUser: 1000, runAsGroup: 1000, fsGroup: 1000}
  containers:
    - name: t
      image: busybox:1.36
      command: [sh, -c, "mkdir -p /nfs/_migration && cd /config && tar -cf /nfs/_migration/$a.tar . && ls -l /nfs/_migration/$a.tar && tar -tf /nfs/_migration/$a.tar | wc -l"]
      volumeMounts: [{name: config, mountPath: /config}, {name: nfs, mountPath: /nfs}]
  volumes:
    - {name: config, persistentVolumeClaim: {claimName: $a}}
    - {name: nfs, nfs: {server: kl-san-1.localdomain, path: /volume1/data}}
EOF
done
for a in sonarr radarr prowlarr qui qbittorrent; do kubectl -n downloads wait pod/stage-$a --for=jsonpath='{.status.phase}'=Succeeded --timeout=300s >/dev/null && echo "== $a" && kubectl -n downloads logs stage-$a; done
```

Expected: each pod ends `Succeeded`; sizes ≈ sonarr 43 M, radarr 58 M, prowlarr 56 M, qui 3.5 M, qbittorrent 16 M (tar sizes slightly larger); file counts > 0. If a pod stays `Pending` with a Longhorn "volume is attached to another node" event, wait — the detach from the old app node takes up to a minute.

- [ ] **Step 3: Verify the tars are readable from the new cluster (proves the NFS path end-to-end) and clean up the helper pods**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl run stagecheck --rm -i --restart=Never --image=busybox:1.36 --overrides='{"spec":{"containers":[{"name":"c","image":"busybox:1.36","command":["sh","-c","ls -l /nfs/_migration && for f in /nfs/_migration/*.tar; do tar -tf $f >/dev/null && echo \"ok $f\"; done"],"volumeMounts":[{"name":"nfs","mountPath":"/nfs"}]}],"volumes":[{"name":"nfs","nfs":{"server":"kl-san-1.localdomain","path":"/volume1/data"}}]}}' 2>&1 | grep -vE '^pod |warning|If you'
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig && kubectl -n downloads delete pod stage-sonarr stage-radarr stage-prowlarr stage-qui stage-qbittorrent
```

Expected: five `ok` lines from the new cluster; helper pods deleted. **Do not proceed to Task 3 until all five are `ok`.**

### Task 3: Remove the namespace from the old cluster (`main` PR, merged first)

- [ ] **Step 1: Branch, delete, PR**

```bash
cd ~/repo/kichi-org/home-ops
git stash push -q -- .gitignore 2>/dev/null || true
git checkout -b chore/downloads-cutover main
git rm -r -q kubernetes/apps/downloads
git commit -q -m "chore(downloads): move to the new cluster"
git push -u origin chore/downloads-cutover
git checkout -q main && git stash pop -q 2>/dev/null || true
```

GitHub MCP `create_pull_request` (head `chore/downloads-cutover`, base `main`), body: "Phase 4 — removes the downloads namespace from the old cluster (configs already staged on the NAS). `v2` PR #PR_V2 enables it on talos-11 afterwards. Merge this one first." Record as `PR_MAIN`.

- [ ] **Step 2: Calvin merges `PR_MAIN`; watch the prune (suspended HelmReleases still get uninstalled by prune because the Kustomization deletes the HelmRelease objects)**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
flux --namespace flux-system reconcile kustomization flux-system --with-source | tail -1
until ! kubectl -n downloads get helmrelease --no-headers 2>/dev/null | grep -q .; do sleep 10; done; echo "helmreleases gone $(date +%T)"
kubectl -n downloads get pvc,svc --no-headers 2>&1 | head
kubectl -n downloads get svc qbittorrent-bittorrent 2>&1 | tail -1
```

Expected: HelmReleases gone; PVCs deleted with them (the tars on the NAS are now the only copy — that is why Task 2 Step 3 gates this); `qbittorrent-bittorrent` Service NotFound → `.123` is no longer announced. If a HelmRelease lingers in `Uninstalling` because it is suspended, `flux -n downloads resume helmrelease <app>` and it finishes.

### Task 4: Enable on the new cluster and restore the configs (merge `PR_V2`)

- [ ] **Step 1: Calvin merges `PR_V2`; wait for the apps to come up empty**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
git checkout -q v2 && git pull -q --ff-only && git branch -D feat/downloads
flux --namespace flux-system reconcile kustomization flux-system --with-source | tail -1
flux -n kube-system reconcile kustomization cilium | tail -1
until [ "$(kubectl -n downloads get helmrelease --no-headers 2>/dev/null | grep -c ' True ')" = "6" ]; do sleep 15; done; echo "six helmreleases ready $(date +%T)"
kubectl -n downloads get pods,pvc --no-headers | awk '{print $1,$2,$3}'
kubectl -n downloads get svc qbittorrent-bittorrent -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
kubectl -n downloads get externalsecret --no-headers | awk '{print $1,$5,$6}'
```

Expected: five deployments Running (fresh config), five PVCs Bound on `openebs-hostpath`, recyclarr CronJob created; `qbittorrent-bittorrent` on `172.16.0.123`; all ExternalSecrets `SecretSynced True`.

- [ ] **Step 2: Scale to 0 and restore each config from the staged tar (wipe first, then untar)**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
for a in sonarr radarr prowlarr qui qbittorrent; do flux -n downloads suspend helmrelease $a; kubectl -n downloads scale deploy/$a --replicas=0; done
until ! kubectl -n downloads get pods --no-headers | grep -vE 'recyclarr|restore-' | grep -q .; do sleep 5; done
for a in sonarr radarr prowlarr qui qbittorrent; do
kubectl -n downloads apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata: {name: restore-$a}
spec:
  restartPolicy: Never
  securityContext: {runAsUser: 1000, runAsGroup: 1000, fsGroup: 1000}
  containers:
    - name: t
      image: busybox:1.36
      command: [sh, -c, "cd /config && rm -rf ./* ./.[!.]* 2>/dev/null; tar -xf /nfs/_migration/$a.tar -C /config && du -sh /config && ls /config | head -5"]
      volumeMounts: [{name: config, mountPath: /config}, {name: nfs, mountPath: /nfs}]
  volumes:
    - {name: config, persistentVolumeClaim: {claimName: $a}}
    - {name: nfs, nfs: {server: kl-san-1.localdomain, path: /volume1/data}}
EOF
done
for a in sonarr radarr prowlarr qui qbittorrent; do kubectl -n downloads wait pod/restore-$a --for=jsonpath='{.status.phase}'=Succeeded --timeout=300s >/dev/null && echo "== $a" && kubectl -n downloads logs restore-$a; done
kubectl -n downloads delete pod restore-sonarr restore-radarr restore-prowlarr restore-qui restore-qbittorrent
for a in sonarr radarr prowlarr qui qbittorrent; do flux -n downloads resume helmrelease $a; done
kubectl -n downloads get pods --no-headers | awk '{print $1,$2,$3}'
```

Expected: each restore prints the old size and familiar files (`config.xml`, `*.db` for *arr; `qBittorrent/` for qbittorrent; `qui.db`/`config.toml` for qui); resuming the HelmReleases scales the deployments back to 1 (Helm reconciles `replicas: 1`); if a deployment stays at 0, `kubectl -n downloads scale deploy/$a --replicas=1`.

- [ ] **Step 3: Functional checks**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
for a in sonarr radarr prowlarr qui qbittorrent; do kubectl -n downloads rollout status deploy/$a --timeout=300s | tail -1; done
for a in sonarr radarr prowlarr qui qbittorrent; do curl -sk --max-time 10 --resolve $a.kichi.live:443:172.16.0.31 -o /dev/null -w "$a http=%{http_code}\n" https://$a.kichi.live/; done
KEY=$(kubectl -n downloads get secret sonarr-secret -o jsonpath='{.data.SONARR__AUTH__APIKEY}' | base64 -d)
curl -sk --resolve sonarr.kichi.live:443:172.16.0.31 -H "X-Api-Key: $KEY" https://sonarr.kichi.live/api/v3/series | jq 'length'
curl -sk --resolve sonarr.kichi.live:443:172.16.0.31 -H "X-Api-Key: $KEY" https://sonarr.kichi.live/api/v3/rootfolder | jq -r '.[] | "\(.path) accessible=\(.accessible)"'
KEY=$(kubectl -n downloads get secret radarr-secret -o jsonpath='{.data.RADARR__AUTH__APIKEY}' | base64 -d)
curl -sk --resolve radarr.kichi.live:443:172.16.0.31 -H "X-Api-Key: $KEY" https://radarr.kichi.live/api/v3/movie | jq 'length'
KEY=$(kubectl -n downloads get secret prowlarr-secret -o jsonpath='{.data.PROWLARR__AUTH__APIKEY}' | base64 -d)
curl -sk --resolve prowlarr.kichi.live:443:172.16.0.31 -H "X-Api-Key: $KEY" https://prowlarr.kichi.live/api/v1/indexer | jq 'length'
kubectl -n downloads exec deploy/qbittorrent -c app -- sh -c 'wget -qO- http://localhost:80/api/v2/torrents/info' | jq 'length'
nc -z -w2 172.16.0.123 50469 && echo "qbittorrent .123:50469 open"
```

Expected: five `http=200/302`; Sonarr series count and Radarr movie count > 0 (library restored), root folders `accessible=true` (NFS `/data`); Prowlarr indexers > 0; qBittorrent lists its torrents; `.123:50469` open (UniFi forward unchanged). If the `*_API_KEY` from 1Password differs from the one in the restored `config.xml`, the `*ARR__AUTH__APIKEY` env var wins and the API calls above still work — then check the Prowlarr → Sonarr/Radarr app links in Prowlarr's UI match (they use the same keys).

- [ ] **Step 4: VolSync first sync**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl -n downloads get replicationsource --no-headers | awk '{print $1,$2,$3}'
for a in sonarr radarr prowlarr qui qbittorrent; do kubectl -n downloads patch replicationsource $a --type merge -p '{"spec":{"trigger":{"manual":"initial"}}}' >/dev/null; done
until [ "$(kubectl -n downloads get replicationsource -o jsonpath='{range .items[*]}{.status.lastManualSync}{"\n"}{end}' | grep -c initial)" = "5" ]; do sleep 15; done
kubectl -n downloads get replicationsource -o jsonpath='{range .items[*]}{.metadata.name} {.status.latestMoverStatus.result} {.status.lastSyncTime}{"\n"}{end}'
```

Expected: five `Successful` syncs; R2 now holds `volsync/{sonarr,radarr,prowlarr,qui,qbittorrent}/`.

- [ ] **Step 5: Recyclarr dry run**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl -n downloads create job --from=cronjob/recyclarr recyclarr-manual && kubectl -n downloads wait job/recyclarr-manual --for=condition=complete --timeout=300s && kubectl -n downloads logs job/recyclarr-manual --tail=15
kubectl -n downloads delete job recyclarr-manual
```

Expected: recyclarr syncs custom formats/quality profiles to the new Sonarr/Radarr without errors.

### Task 5: Close out

- [ ] **Step 1: Remove the staging directory from the NAS**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl run stageclean --rm -i --restart=Never --image=busybox:1.36 --overrides='{"spec":{"securityContext":{"runAsUser":1000,"fsGroup":1000},"containers":[{"name":"c","image":"busybox:1.36","command":["sh","-c","rm -rf /nfs/_migration && ls /nfs | grep -c _migration || echo staging removed"],"volumeMounts":[{"name":"nfs","mountPath":"/nfs"}]}],"volumes":[{"name":"nfs","nfs":{"server":"kl-san-1.localdomain","path":"/volume1/data"}}]}}' 2>&1 | grep -vE '^pod |warning|If you'
```

Expected: `staging removed`. (Only after Task 4 Step 4 shows VolSync copies in R2.)

- [ ] **Step 2: Execution log** on `v2` and `main` (`docs(plan): phase 4 execution log`), push both (background the pushes).
- [ ] **Step 3: Memory** — `migration-progress`: phase 4 done, outage window, next = step 5 (Dispatcharr).
- [ ] **Step 4: Rollback (only if needed, before Task 5 Step 1)** — revert `PR_V2` (new apps + `.123` gone), revert `PR_MAIN` (old apps reinstall with empty PVCs), then rerun Task 4 Step 2's restore loop against the **old** cluster to reload the staged tars.

---

## Execution log

- PR_V2 / PR_MAIN: #778 (`v2`, merged b9b402d) / #779 (`main`, merged 4ed1aac); follow-up `bf20d5f fix(recyclarr): use an emptydir for the config cache`
- Outage window: old stopped 22:25:00 → new restored and scaled up ≈22:57 (+08)
- Staged tar sizes: sonarr 43.8 MB / radarr 59.3 MB / prowlarr 57.3 MB / qui 3.5 MB / qbittorrent 11.7 MB; verified readable from talos-11 before the `main` merge
- Library counts after restore: 10 series (2 root folders accessible) / 20 movies (root accessible) / 5 indexers / 4 torrents; `.123:50469` open; five VolSync first syncs Successful 14:58Z; recyclarr manual run synced sonarr+radarr
- Date completed: 2026-08-30

Deviations:

- Deleting the `main` Kustomizations while the HelmReleases were **suspended** removed the HelmRelease objects without running `helm uninstall` — Deployments, Services (incl. `.123`) and PVCs were orphaned. Fixed by deleting the whole `downloads` namespace on the old cluster. Rule: resume HelmReleases before the prune, or plan to delete the namespace.
- After `flux resume`, Helm did not scale the deployments back from 0 (no drift correction) — `kubectl scale --replicas=1` was needed.
- recyclarr is a CronJob, so its `WaitForFirstConsumer` PVC never bound and the Helm install timed out; its config is a regenerable cache → switched to `emptyDir` (no VolSync for recyclarr, as planned).
