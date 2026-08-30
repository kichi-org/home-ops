# Single-node migration — Phase 7 (DNS + gateway IPs) implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move Technitium (LAN DNS + ad-blocking, `172.16.0.200`) and its external-dns (`technitium-dns`, RFC2136) to `talos-11`, and re-pin the new cluster's gateways to their canonical addresses — k8s-gateway `.11`, envoy-internal `.12`, envoy-external `.13` — so every `*.kichi.live` name on the LAN resolves to the new cluster. After this the old cluster serves nothing user-facing.

**Architecture:** Two-PR pattern. `main` removes Technitium, technitium-dns, k8s-gateway and envoy-gateway (releasing `.200/.11/.12/.13`); `v2` then adds Technitium + technitium-dns and changes the three gateway pins from `.31/.32/.33` to `.12/.11/.13` (Cilium re-assigns the LoadBalancer IPs in place). Technitium's `/etc/dns` (754 MB, root-owned: zones, block lists, apps, stats) is tarred to NFS staging before the `main` merge and restored after the `v2` merge. The `kichi.live` zone data inside it already points `internal → .12` / `external → .13`, so no record rewrite is needed. DHCP already hands out `.200` (+ `192.168.85.1` fallback) and the UCG NS record already points at `.11` — no UniFi change.

**Tech Stack:** Flux, app-template 5.1.0, `technitium/dns-server:15.4.0`, external-dns 1.21.1 (rfc2136), k8s-gateway 3.7.2, envoy-gateway 1.9.1, Cilium LB-IPAM + L2, OpenEBS hostpath, VolSync (privileged mover for root-owned data), ESO/1Password item `technitium`.

**Spec:** `docs/superpowers/specs/2026-08-30-single-node-migration-design.md` (§3.3, §7, §8 step 7, §9) + `network-design-decisions` #4.

## Global constraints

- `.200`, `.11`, `.12`, `.13` are each announced by one cluster at a time (spec §9); the `v2` PR is merged only after the `main` prune has released them.
- LAN impact: while `.200`/`.11` are down, clients fall back to `192.168.85.1` for the internet and `*.kichi.live` names do not resolve. Keep the gap short: stage → merge `main` → merge `v2` back-to-back.
- No UniFi/Cloudflare changes. Commits: subject only. Tooling as in the earlier plans (`mise` shims, `mise exec -- git commit` on `v2`).
- Resume suspended HelmReleases before the prune (phase 4 lesson); capture data before the `main` merge (phase 3/4 lesson).

---

## File map

Branch `v2`:

| Path                                                       | Responsibility                                                                             |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `kubernetes/apps/network/technitium/{ks.yaml,app/*}`       | copied; `openebs-hostpath` + `retain: true`; VolSync component with `VOLSYNC_UID/GID: "0"` |
| `kubernetes/apps/network/technitium-dns/{ks.yaml,app/*}`   | copied verbatim (owner `technitium`, TSIG from 1Password)                                  |
| `kubernetes/apps/network/namespace.yaml`                   | + annotation `volsync.backube/privileged-movers: "true"` (root-owned Technitium files)     |
| `kubernetes/apps/network/kustomization.yaml`               | + technitium, technitium-dns                                                               |
| `kubernetes/apps/network/k8s-gateway/app/helmrelease.yaml` | `lbipam.cilium.io/ips: 172.16.0.11`                                                        |
| `kubernetes/apps/network/envoy-gateway/app/envoy.yaml`     | internal `172.16.0.12`, external `172.16.0.13`                                             |
| `kubernetes/apps/kube-system/cilium/app/networks.yaml`     | + blocks `.11–.13`, `.200`                                                                 |

Branch `main`: delete `kubernetes/apps/network/{technitium,technitium-dns,k8s-gateway,envoy-gateway}/` and their lines in `network/kustomization.yaml`.

Scratch: `/volume1/data/_migration/technitium.tar`.

---

### Task 1: Port to `v2` on `feat/dns` (PR not merged yet)

- [ ] **Step 1: Copy, rewrite, re-pin, extend pool, validate**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH
git checkout -b feat/dns v2
cp -R ../home-ops/kubernetes/apps/network/technitium kubernetes/apps/network/technitium
cp -R ../home-ops/kubernetes/apps/network/technitium-dns kubernetes/apps/network/technitium-dns
cp ../home-ops/docs/superpowers/plans/2026-08-31-single-node-migration-phase-7-dns.md docs/superpowers/plans/
python3 - <<'EOF'
import pathlib
p=pathlib.Path('kubernetes/apps/network/technitium/ks.yaml'); s=p.read_text()
old='spec:\n  interval: 1h\n'
assert old in s
s=s.replace(old,'spec:\n  components:\n    - ../../../../components/volsync\n  dependsOn:\n    - name: openebs\n      namespace: openebs-system\n    - name: onepassword\n      namespace: external-secrets\n  interval: 1h\n',1)
s=s.replace('  targetNamespace: network\n','  targetNamespace: network\n  postBuild:\n    substitute:\n      APP: technitium\n      VOLSYNC_UID: "0"\n      VOLSYNC_GID: "0"\n',1)
p.write_text(s)
p=pathlib.Path('kubernetes/apps/network/technitium/app/helmrelease.yaml'); s=p.read_text()
old='        type: persistentVolumeClaim\n        storageClass: longhorn\n'
assert old in s
p.write_text(s.replace(old,'        type: persistentVolumeClaim\n        storageClass: openebs-hostpath\n        retain: true\n'))
p=pathlib.Path('kubernetes/apps/network/technitium-dns/ks.yaml'); s=p.read_text()
if 'dependsOn' not in s:
    s=s.replace('spec:\n  interval: 1h\n','spec:\n  dependsOn:\n    - name: technitium\n    - name: onepassword\n      namespace: external-secrets\n  interval: 1h\n',1)
p.write_text(s)
p=pathlib.Path('kubernetes/apps/network/namespace.yaml'); s=p.read_text()
if 'privileged-movers' not in s:
    s=s.replace('metadata:\n  name: network\n','metadata:\n  name: network\n  annotations:\n    volsync.backube/privileged-movers: "true"\n',1)
p.write_text(s)
p=pathlib.Path('kubernetes/apps/network/kustomization.yaml'); s=p.read_text()
assert 'technitium' not in s
p.write_text(s.replace('  - ./k8s-gateway/ks.yaml\n','  - ./k8s-gateway/ks.yaml\n  - ./technitium/ks.yaml\n  - ./technitium-dns/ks.yaml\n'))
p=pathlib.Path('kubernetes/apps/network/k8s-gateway/app/helmrelease.yaml'); s=p.read_text()
assert '172.16.0.32' in s; p.write_text(s.replace('172.16.0.32','172.16.0.11'))
p=pathlib.Path('kubernetes/apps/network/envoy-gateway/app/envoy.yaml'); s=p.read_text()
assert '172.16.0.31' in s and '172.16.0.33' in s
p.write_text(s.replace('172.16.0.31','172.16.0.12').replace('172.16.0.33','172.16.0.13'))
p=pathlib.Path('kubernetes/apps/kube-system/cilium/app/networks.yaml'); s=p.read_text()
old='    - start: "172.16.0.30"\n      stop: "172.16.0.99"\n'
assert old in s
p.write_text(s.replace(old,'    - start: "172.16.0.11"\n      stop: "172.16.0.13"\n'+old+'    - start: "172.16.0.200"\n      stop: "172.16.0.200"\n'))
EOF
grep -n 'privileged' kubernetes/apps/network/namespace.yaml; grep -rn '172.16.0.3[123]' kubernetes/apps/network || echo "no transition IPs left"
for d in network network/technitium/app network/technitium-dns/app network/k8s-gateway/app network/envoy-gateway/app kube-system/cilium/app; do kubectl kustomize kubernetes/apps/$d >/dev/null && echo "OK $d"; done
```

Expected: annotation present; no `.31/.32/.33` left; all builds OK. Note: the `network` namespace object is Flux-managed from `namespace.yaml`, so the annotation applies on reconcile.

- [ ] **Step 2: Commit, push, PR (base `v2`)** — `feat(network): port technitium and re-pin gateways to canonical ips`. Record `PR_V2`. **Do not merge yet.**

### Task 2: Stage Technitium's data (LAN DNS outage starts here)

- [ ] **Step 1: Freeze, tar, verify**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
flux -n network suspend helmrelease technitium && kubectl -n network scale deploy/technitium --replicas=0
until ! kubectl -n network get pods -l app.kubernetes.io/name=technitium --no-headers | grep -q .; do sleep 5; done; echo "old technitium stopped $(date +%T)"
kubectl -n network apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: stage-technitium}
spec:
  restartPolicy: Never
  containers:
    - name: t
      image: busybox:1.36
      command: [sh, -c, "mkdir -p /nfs/_migration && cd /config && tar -cf /nfs/_migration/technitium.tar --exclude=./stats --exclude=./logs --exclude=./cache.bin --exclude=./lost+found . && ls -l /nfs/_migration/technitium.tar && tar -tf /nfs/_migration/technitium.tar | wc -l"]
      volumeMounts: [{name: config, mountPath: /config}, {name: nfs, mountPath: /nfs}]
  volumes:
    - {name: config, persistentVolumeClaim: {claimName: technitium}}
    - {name: nfs, nfs: {server: kl-san-1.localdomain, path: /volume1/data}}
EOF
kubectl -n network wait pod/stage-technitium --for=jsonpath='{.status.phase}'=Succeeded --timeout=600s && kubectl -n network logs stage-technitium && kubectl -n network delete pod stage-technitium
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl run stagecheck --rm -i --restart=Never --image=busybox:1.36 --overrides='{"spec":{"containers":[{"name":"c","image":"busybox:1.36","command":["sh","-c","tar -tf /nfs/_migration/technitium.tar | grep -cE \"^./(zones|dns.config|blocked.config|allowed.config|apps|blocklists)\" && echo ok technitium.tar"],"volumeMounts":[{"name":"nfs","mountPath":"/nfs"}]}],"volumes":[{"name":"nfs","nfs":{"server":"kl-san-1.localdomain","path":"/volume1/data"}}]}}' 2>&1 | grep -vE '^pod |warning|If you|recorded'
```

Expected: tar ≈ 120 MB (stats/logs/cache excluded — regenerated), contains `zones/`, `dns.config`, block lists, `apps/`; `ok technitium.tar` from talos-11. The stage pod runs as root (no securityContext) because the files are root-owned.

### Task 3: Remove DNS + gateways from the old cluster (`main` PR, merged first)

- [ ] **Step 1: Resume, branch, delete, PR**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
flux -n network resume helmrelease technitium >/dev/null
git stash push -q -- .gitignore 2>/dev/null || true
git checkout -b chore/dns-cutover main
git rm -r -q kubernetes/apps/network/technitium kubernetes/apps/network/technitium-dns kubernetes/apps/network/k8s-gateway kubernetes/apps/network/envoy-gateway
sed -i '' -e '/^  - \.\/technitium\/ks\.yaml$/d' -e '/^  - \.\/technitium-dns\/ks\.yaml$/d' -e '/^  - \.\/k8s-gateway\/ks\.yaml$/d' -e '/^  - \.\/envoy-gateway\/ks\.yaml$/d' kubernetes/apps/network/kustomization.yaml
git add kubernetes && git commit -q -m "chore(network): move dns and gateways to the new cluster"
git push -u origin chore/dns-cutover
git checkout -q main && git stash pop -q 2>/dev/null || true
```

GitHub MCP `create_pull_request` (base `main`). Record `PR_MAIN`. (The remaining old-cluster HTTPRoutes — grafana, prometheus, alertmanager, longhorn, victoria-logs, flux-webhook — lose their gateway; expected, the old cluster is retired next.)

- [ ] **Step 2: Calvin merges; watch the prune and the IPs**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
flux --namespace flux-system reconcile kustomization flux-system --with-source | tail -1
until ! kubectl -n network get helmrelease --no-headers 2>/dev/null | grep -E 'technitium|k8s-gateway|envoy-gateway' | grep -q .; do sleep 10; done; echo "pruned $(date +%T)"
kubectl get svc -A --no-headers | grep -E '172\.16\.0\.(200|11|12|13)\b' || echo "canonical IPs released"
```

If a HelmRelease lingers (suspended at deletion), delete the leftover `network` objects by hand (`kubectl -n network delete deploy,svc,pvc --all` is acceptable — the tar is the copy).

### Task 4: Enable on the new cluster and restore (merge `PR_V2`)

- [ ] **Step 1: Merge, reconcile, watch the IPs move, restore Technitium**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
git checkout -q v2 && git pull -q --ff-only && git branch -D feat/dns
flux --namespace flux-system reconcile kustomization flux-system --with-source | tail -1
flux -n kube-system reconcile kustomization cilium | tail -1; flux -n network reconcile kustomization envoy-gateway | tail -1; flux -n network reconcile kustomization k8s-gateway | tail -1
until [ "$(kubectl -n network get svc k8s-gateway envoy-internal envoy-external -o jsonpath='{range .items[*]}{.status.loadBalancer.ingress[0].ip} {end}')" = "172.16.0.11 172.16.0.12 172.16.0.13 " ]; do sleep 10; done; echo "gateways re-pinned $(date +%T)"
until kubectl -n network get helmrelease technitium -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null | grep -q True; do sleep 15; done
kubectl -n network get svc technitium-dns -o jsonpath='technitium LB={.status.loadBalancer.ingress[0].ip}{"\n"}'
flux -n network suspend helmrelease technitium && kubectl -n network scale deploy/technitium --replicas=0
until ! kubectl -n network get pods -l app.kubernetes.io/name=technitium --no-headers | grep -q .; do sleep 5; done
kubectl -n network apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: restore-technitium}
spec:
  restartPolicy: Never
  containers:
    - name: t
      image: busybox:1.36
      command: [sh, -c, "cd /config && rm -rf ./* ./.[!.]* 2>/dev/null; tar -xf /nfs/_migration/technitium.tar -C /config && du -sh /config && ls /config | tr '\\n' ' '"]
      volumeMounts: [{name: config, mountPath: /config}, {name: nfs, mountPath: /nfs}]
  volumes:
    - {name: config, persistentVolumeClaim: {claimName: technitium}}
    - {name: nfs, nfs: {server: kl-san-1.localdomain, path: /volume1/data}}
EOF
kubectl -n network wait pod/restore-technitium --for=jsonpath='{.status.phase}'=Succeeded --timeout=600s && kubectl -n network logs restore-technitium && kubectl -n network delete pod restore-technitium
flux -n network resume helmrelease technitium >/dev/null; kubectl -n network scale deploy/technitium --replicas=1; kubectl -n network rollout status deploy/technitium --timeout=300s | tail -1; echo "technitium up $(date +%T)"
```

Expected: the three gateway Services on `.11/.12/.13` (Cilium updates them in place), `technitium LB=172.16.0.200`, restore lists `zones dns.config blocked.config …`, Technitium rolls out. Record the time (end of the LAN DNS gap).

- [ ] **Step 2: Verify resolution from the LAN and the new cluster's split-DNS**

```bash
dig +short @172.16.0.200 grafana.kichi.live plex.kichi.live sonarr.kichi.live teslamate.kichi.live | tr '\n' ' '; echo
dig +short @172.16.0.11 grafana.kichi.live; dig +short @172.16.0.200 google.com | head -1
dig +short @172.16.0.200 doubleclick.net | head -1   # ad-block: expect 0.0.0.0 / empty
dig +short grafana.kichi.live   # via the LAN resolver (DHCP → .200)
curl -sk --max-time 10 -o /dev/null -w 'grafana via name http=%{http_code}\n' https://grafana.kichi.live/api/health
curl -sk --max-time 10 -o /dev/null -w 'plex via name http=%{http_code}\n' https://plex.kichi.live/identity
curl -sk --max-time 10 -o /dev/null -w 'technitium ui http=%{http_code}\n' https://technitium.kichi.live/
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl -n network logs deploy/external-dns-technitium --since=10m | grep -iE 'Changing|error' | head -6
```

Expected: `*.kichi.live` → `172.16.0.12` (internal CNAME) / `.13` for `plex`, `external`; k8s-gateway on `.11` answers; upstream resolution works; ad-block returns nothing/0.0.0.0; via the LAN resolver `grafana.kichi.live` → `.12` and HTTPS 200 from the **new** Grafana (13.0.1); Plex 200 via name (the phase-6 LAN 404 is gone); technitium-dns logs show no errors (records already match, so few/no changes).

- [ ] **Step 3: VolSync first sync (privileged mover), staging cleanup**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl -n network patch replicationsource technitium --type merge -p '{"spec":{"trigger":{"manual":"initial"}}}'
until [ "$(kubectl -n network get replicationsource technitium -o jsonpath='{.status.lastManualSync}')" = "initial" ]; do sleep 20; done
kubectl -n network get replicationsource technitium -o jsonpath='{.status.latestMoverStatus.result} {.status.lastSyncTime}{"\n"}'
kubectl run stageclean --rm -i --restart=Never --image=busybox:1.36 --overrides='{"spec":{"containers":[{"name":"c","image":"busybox:1.36","command":["sh","-c","rm -rf /nfs/_migration && echo staging removed"],"volumeMounts":[{"name":"nfs","mountPath":"/nfs"}]}],"volumes":[{"name":"nfs","nfs":{"server":"kl-san-1.localdomain","path":"/volume1/data"}}]}}' 2>&1 | grep -vE '^pod |warning|If you|recorded'
```

Expected: `Successful` (if the mover fails with a permission error, the namespace annotation did not apply — `kubectl annotate ns network volsync.backube/privileged-movers=true --overwrite` and retrigger).

### Task 5: Close out

- [ ] Execution log on both branches; memory (`migration-progress`, `network-facts`: canonical IPs now on talos-11; phase 7 done; next = step 8 soak with old VMs off). Rollback: revert `PR_V2` (gateways back to `.31–.33`, technitium gone) then `PR_MAIN` (old comes back on empty PVC → restore from the tar).

---

## Execution log

- PR_V2 / PR_MAIN: _(…)_
- LAN DNS gap: old technitium stopped _(…)_ → new technitium up _(…)_
- Tar size / entries: _(…)_
- Resolution checks: _(…)_
- Date completed: _(…)_
