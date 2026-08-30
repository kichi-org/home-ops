# Single-node migration — Phase 6 (Plex + Cloudflare) implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move Plex (with its 1.4 GB metadata, no rescan), the Cloudflare Tunnel `kubernetes` and the Cloudflare `external-dns` from the old cluster to `talos-11`; keep `plex.kichi.live` on the tunnel; add the direct-stream path (`plex-direct.kichi.live`, WAN 32400 → `172.16.0.128`); drop the `echo` canary.

**Architecture:** Same two-PR pattern (`main` removes first, `v2` enables second) because the tunnel and the external-dns owner `default` are singletons: two `cloudflared` on one tunnel would load-balance between clusters, and two external-dns with the same owner would fight over records. The Plex config PVC is Helm-owned on the old cluster, so it is tarred to NFS staging before the `main` merge and restored onto the new hostpath PVC after the `v2` merge. The Cloudflare/UniFi prep (grey `plex-direct` A record, DDNS repoint, 32400 forward) is done *before* the cutover — `.128` is the same on both clusters, so it is safe and testable against the old Plex.

**Tech Stack:** Flux, app-template 5.1.0, `ghcr.io/home-operations/plex:1.43.3`, cloudflared 2026.8.2, external-dns 1.21.1 (cloudflare provider), SOPS/age (existing secrets reused), OpenEBS hostpath, VolSync, Cilium LB-IPAM, Cloudflare API (MCP), UniFi API (MCP, preview → confirm).

**Spec:** `docs/superpowers/specs/2026-08-30-single-node-migration-design.md` (§3.3, §3.4, §7, §8 step 6, §9) + `network-design-decisions` #5/#6.

## Global constraints

- One cluster runs `cloudflared` for tunnel `9c7a341c-700f-4141-916d-ded983d0b1ae` and one runs external-dns owner `default` at any time (spec §9).
- `.128` is announced by one cluster at a time; the `v2` pool gets a `.128` block in the `v2` PR.
- Cloudflare mail/iCloud records and the `mytunnel_iph` hostnames are never touched. Every Cloudflare non-GET and every UniFi mutation is previewed and approved by Calvin before running.
- Capture everything needed from the old cluster **before** merging the `main` PR; **resume** suspended HelmReleases before the prune (phase 4 lesson) — or delete the namespace by hand afterwards.
- Commits: subject only, no body, no trailers. New cluster tooling: `cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig SOPS_AGE_KEY_FILE=$PWD/age.key`; commit with `mise exec -- git commit`. Old cluster: `cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig`.

---

## File map

Branch `v2`:

| Path | Responsibility |
|---|---|
| `kubernetes/apps/media/{namespace,kustomization}.yaml` | namespace (`alerts` component), `./plex/ks.yaml` |
| `kubernetes/apps/media/plex/ks.yaml` | deps `openebs`, `onepassword`; VolSync component, `APP: plex` |
| `kubernetes/apps/media/plex/app/*` | copied; `openebs-hostpath` + `retain: true`; `PLEX_ADVERTISE_URL` gains `https://plex-direct.kichi.live:32400` |
| `kubernetes/apps/network/cloudflare-tunnel/{ks.yaml,app/*}` | copied verbatim from `main` incl. `secret.sops.yaml` (tunnel token) and the `external.kichi.live` DNSEndpoint |
| `kubernetes/apps/network/cloudflare-dns/{ks.yaml,app/*}` | copied verbatim incl. `secret.sops.yaml` (API token), owner `default` |
| `kubernetes/apps/network/kustomization.yaml` | + `cloudflare-dns/ks.yaml`, `cloudflare-tunnel/ks.yaml` |
| `kubernetes/apps/kube-system/cilium/app/networks.yaml` | + block `172.16.0.128–172.16.0.128` |

Branch `main`: delete `kubernetes/apps/media/plex/`, `kubernetes/apps/network/cloudflare-tunnel/`, `kubernetes/apps/network/cloudflare-dns/`, `kubernetes/apps/default/echo/`; drop their lines from the three `kustomization.yaml`s.

Scratch: `/volume1/data/_migration/plex.tar` on the NAS.

---

### Task 1: Port Plex + Cloudflare apps to `v2` (PR not merged yet)

- [ ] **Step 1: Branch, copy, rewrite**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH SOPS_AGE_KEY_FILE=$PWD/age.key
git checkout -b feat/plex-cloudflare v2
mkdir -p kubernetes/apps/media && cp ../home-ops/kubernetes/apps/media/namespace.yaml kubernetes/apps/media/
cp -R ../home-ops/kubernetes/apps/media/plex kubernetes/apps/media/plex
cp -R ../home-ops/kubernetes/apps/network/cloudflare-tunnel kubernetes/apps/network/cloudflare-tunnel
cp -R ../home-ops/kubernetes/apps/network/cloudflare-dns kubernetes/apps/network/cloudflare-dns
cp ../home-ops/docs/superpowers/plans/2026-08-30-single-node-migration-phase-6-plex-cloudflare.md docs/superpowers/plans/
cat > kubernetes/apps/media/kustomization.yaml <<'EOF'
---
# yaml-language-server: $schema=https://json.schemastore.org/kustomization
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: media
components:
  - ../../components/alerts
resources:
  - ./namespace.yaml
  - ./plex/ks.yaml
EOF
python3 - <<'EOF'
import pathlib
p=pathlib.Path('kubernetes/apps/media/plex/ks.yaml'); s=p.read_text()
old='  dependsOn:\n    - name: longhorn\n      namespace: longhorn-system\n'
assert old in s
p.write_text(s.replace(old,'  components:\n    - ../../../../components/volsync\n  dependsOn:\n    - name: openebs\n      namespace: openebs-system\n    - name: onepassword\n      namespace: external-secrets\n'))
p=pathlib.Path('kubernetes/apps/media/plex/app/helmrelease.yaml'); s=p.read_text()
old='        type: persistentVolumeClaim\n        storageClass: longhorn\n'
assert old in s
s=s.replace(old,'        type: persistentVolumeClaim\n        storageClass: openebs-hostpath\n        retain: true\n').replace('        # existingClaim: "{{ .Release.Name }}"\n','')
old='PLEX_ADVERTISE_URL: https://plex.kichi.live:443,http://172.16.0.128:32400,http://plex.media.svc.cluster.local:32400'
assert old in s
s=s.replace(old,'PLEX_ADVERTISE_URL: https://plex.kichi.live:443,https://plex-direct.kichi.live:32400,http://172.16.0.128:32400,http://plex.media.svc.cluster.local:32400')
p.write_text(s)
p=pathlib.Path('kubernetes/apps/network/kustomization.yaml'); s=p.read_text()
assert 'cloudflare' not in s
p.write_text(s.replace('  - ./envoy-gateway/ks.yaml\n','  - ./cloudflare-dns/ks.yaml\n  - ./cloudflare-tunnel/ks.yaml\n  - ./envoy-gateway/ks.yaml\n'))
p=pathlib.Path('kubernetes/apps/kube-system/cilium/app/networks.yaml'); s=p.read_text()
old='    - start: "172.16.0.123"\n      stop: "172.16.0.123"\n'
assert old in s
p.write_text(s.replace(old, old+'    - start: "172.16.0.128"\n      stop: "172.16.0.128"\n'))
EOF
sops decrypt kubernetes/apps/network/cloudflare-tunnel/app/secret.sops.yaml | yq -r '.stringData | keys | join(",")'
sops decrypt kubernetes/apps/network/cloudflare-dns/app/secret.sops.yaml | yq -r '.stringData | keys | join(",")'
for d in media media/plex/app network network/cloudflare-tunnel/app network/cloudflare-dns/app kube-system/cilium/app; do kubectl kustomize kubernetes/apps/$d >/dev/null && echo "OK $d"; done
grep -rn 'longhorn' kubernetes/apps/media kubernetes/apps/network/cloudflare-* || echo "no longhorn refs"
```
Expected: `TUNNEL_TOKEN` and `api-token` decrypt with the v2 `age.key` (same recipient); all builds OK; no longhorn refs. The tunnel's `grafanadashboard.yaml` needs the Grafana CRDs — present on `v2`.

- [ ] **Step 2: Commit, push, open PR (base `v2`)** — `feat(media): port plex, cloudflare tunnel and external-dns`. Record `PR_V2`. **Do not merge yet.**

### Task 2: Cloudflare + UniFi prep (safe before the cutover — `.128` is unchanged)

- [ ] **Step 1: Cloudflare — create the grey A record `plex-direct.kichi.live` = current WAN IP** (MCP `execute`, non-GET → Calvin approves)

```js
async () => cloudflare.request({ method: "POST", path: "/zones/d287b9fab491487559fdee19d7a80d9f/dns_records", body: { type: "A", name: "plex-direct", content: "<current apex A content, 202.186.206.83 at plan time>", ttl: 120, proxied: false, comment: "Plex direct 32400; updated by UniFi DDNS" } })
```
Expected: record created, `proxied:false`. It is not managed by external-dns (no owner TXT), so both external-dns instances leave it alone.

- [ ] **Step 2: UniFi — repoint the Cloudflare DDNS entry from `kichi.live` to `plex-direct.kichi.live`** (`unifi_update_dynamic_dns`, id `6837499f485d90786c7cc7f0`, preview → confirm). Field: `host_name: plex-direct.kichi.live` (login/token unchanged). Expected: entry updated; within minutes the record shows the WAN IP (inadyn updates, doesn't create). The apex `A kichi.live` stays as-is (stale but harmless; delete later if wanted).

- [ ] **Step 3: UniFi — add port forward `Plex` TCP 32400 → `172.16.0.128:32400`** (`unifi_create_simple_port_forward` / `unifi_create_port_forward`, preview → confirm). Verify from outside is not possible from the LAN (hairpin); verify via Plex's Remote Access page later (Task 5).

### Task 3: Freeze old Plex and stage its config on the NAS

- [ ] **Step 1: Suspend + scale to 0, tar `/config` (1.4 GB), verify from the new cluster**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
flux -n media suspend helmrelease plex && kubectl -n media scale deploy/plex --replicas=0
until ! kubectl -n media get pods -l app.kubernetes.io/name=plex --no-headers | grep -q .; do sleep 5; done; echo "old plex stopped $(date +%T)"
kubectl -n media apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: stage-plex}
spec:
  restartPolicy: Never
  securityContext: {runAsUser: 1000, runAsGroup: 1000, fsGroup: 1000}
  containers:
    - name: t
      image: busybox:1.36
      command: [sh, -c, "mkdir -p /nfs/_migration && cd /config && tar -cf /nfs/_migration/plex.tar . && ls -l /nfs/_migration/plex.tar && tar -tf /nfs/_migration/plex.tar | wc -l"]
      volumeMounts: [{name: config, mountPath: /config}, {name: nfs, mountPath: /nfs}]
  volumes:
    - {name: config, persistentVolumeClaim: {claimName: plex}}
    - {name: nfs, nfs: {server: kl-san-1.localdomain, path: /volume1/data}}
EOF
kubectl -n media wait pod/stage-plex --for=jsonpath='{.status.phase}'=Succeeded --timeout=900s && kubectl -n media logs stage-plex && kubectl -n media delete pod stage-plex
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl run stagecheck --rm -i --restart=Never --image=busybox:1.36 --overrides='{"spec":{"containers":[{"name":"c","image":"busybox:1.36","command":["sh","-c","ls -l /nfs/_migration && tar -tf /nfs/_migration/plex.tar >/dev/null && echo ok plex.tar"],"volumeMounts":[{"name":"nfs","mountPath":"/nfs"}]}],"volumes":[{"name":"nfs","nfs":{"server":"kl-san-1.localdomain","path":"/volume1/data"}}]}}' 2>&1 | grep -vE '^pod |warning|If you|recorded'
```
Expected: tar ≈ 1.4 GB, thousands of entries; `ok plex.tar` from talos-11. Record the stop time (start of the Plex outage).

### Task 4: Remove Plex, tunnel, external-dns and echo from the old cluster (`main` PR, merged first)

- [ ] **Step 1: Resume the suspended HelmRelease (so the prune uninstalls cleanly), branch, delete, PR**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
flux -n media resume helmrelease plex >/dev/null   # deployment stays at 0 replicas
git stash push -q -- .gitignore 2>/dev/null || true
git checkout -b chore/plex-cloudflare-cutover main
git rm -r -q kubernetes/apps/media/plex kubernetes/apps/network/cloudflare-tunnel kubernetes/apps/network/cloudflare-dns kubernetes/apps/default/echo
sed -i '' '/^  - \.\/plex\/ks\.yaml$/d' kubernetes/apps/media/kustomization.yaml
sed -i '' -e '/^  - \.\/cloudflare-dns\/ks\.yaml$/d' -e '/^  - \.\/cloudflare-tunnel\/ks\.yaml$/d' kubernetes/apps/network/kustomization.yaml
sed -i '' '/^  - \.\/echo\/ks\.yaml$/d' kubernetes/apps/default/kustomization.yaml
git add kubernetes && git commit -q -m "chore: move plex, cloudflare tunnel and external-dns to the new cluster"
git push -u origin chore/plex-cloudflare-cutover
git checkout -q main && git stash pop -q 2>/dev/null || true
```
GitHub MCP `create_pull_request` (base `main`). Record `PR_MAIN`. Note: if `kubernetes/apps/media/kustomization.yaml` ends up with only `namespace.yaml`, that is fine (namespace stays until step 9).

- [ ] **Step 2: Calvin merges `PR_MAIN`; watch the prune and the tunnel**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
flux --namespace flux-system reconcile kustomization flux-system --with-source | tail -1
until ! kubectl get helmrelease -A --no-headers | grep -E 'plex|cloudflare-tunnel|cloudflare-dns|echo' | grep -q .; do sleep 10; done; echo "pruned $(date +%T)"
kubectl get svc -A --no-headers | grep -c '172.16.0.128' || echo ".128 released"
kubectl -n media get pvc plex 2>&1 | tail -1
```
Cloudflare (GET): `cloudflare.request({method:"GET", path:"/accounts/932a695a6a0d664160ac575fa0e4fda8/cfd_tunnel/9c7a341c-700f-4141-916d-ded983d0b1ae/connections"})` → expected `[]` (no connectors). DNS records `plex`, `flux-webhook`, `external` still exist (external-dns does not clean up on delete); `echo` still exists until the new external-dns removes it. If any HelmRelease is stuck (suspended at deletion), delete the leftover objects by hand as in phase 4.

### Task 5: Enable on the new cluster and restore Plex (merge `PR_V2`)

- [ ] **Step 1: Calvin merges `PR_V2`; wait, then restore the config**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
git checkout -q v2 && git pull -q --ff-only && git branch -D feat/plex-cloudflare
flux --namespace flux-system reconcile kustomization flux-system --with-source | tail -1; flux -n kube-system reconcile kustomization cilium | tail -1
until kubectl -n media get helmrelease plex -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null | grep -q True; do sleep 15; done
kubectl -n network get pods --no-headers | grep -E 'cloudflare|external-dns' | awk '{print $1,$2,$3}'
kubectl -n media get svc plex -o jsonpath='plex LB={.status.loadBalancer.ingress[0].ip}{"\n"}'
flux -n media suspend helmrelease plex && kubectl -n media scale deploy/plex --replicas=0
until ! kubectl -n media get pods -l app.kubernetes.io/name=plex --no-headers | grep -q .; do sleep 5; done
kubectl -n media apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: restore-plex}
spec:
  restartPolicy: Never
  securityContext: {runAsUser: 1000, runAsGroup: 1000, fsGroup: 1000}
  containers:
    - name: t
      image: busybox:1.36
      command: [sh, -c, "cd /config && rm -rf ./* ./.[!.]* 2>/dev/null; tar -xf /nfs/_migration/plex.tar -C /config && du -sh /config && ls '/config/Library/Application Support/Plex Media Server' | head"]
      volumeMounts: [{name: config, mountPath: /config}, {name: nfs, mountPath: /nfs}]
  volumes:
    - {name: config, persistentVolumeClaim: {claimName: plex}}
    - {name: nfs, nfs: {server: kl-san-1.localdomain, path: /volume1/data}}
EOF
kubectl -n media wait pod/restore-plex --for=jsonpath='{.status.phase}'=Succeeded --timeout=900s && kubectl -n media logs restore-plex && kubectl -n media delete pod restore-plex
flux -n media resume helmrelease plex >/dev/null; kubectl -n media scale deploy/plex --replicas=1; kubectl -n media rollout status deploy/plex --timeout=300s | tail -1
```
Expected: cloudflared + external-dns Running, `plex LB=172.16.0.128`, restore prints ≈1.4G with `Metadata`, `Plug-in Support`, `Preferences.xml`; Plex rolls out. Record the time (end of the Plex outage).

- [ ] **Step 2: Verify tunnel, DNS and Plex**

```bash
curl -s --max-time 15 -o /dev/null -w 'plex via tunnel http=%{http_code}\n' https://plex.kichi.live/identity
curl -s --max-time 10 -o /dev/null -w 'plex .128 http=%{http_code}\n' http://172.16.0.128:32400/identity
curl -s --max-time 15 -o /dev/null -w 'flux-webhook via tunnel http=%{http_code}\n' https://flux-webhook.kichi.live/
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
TOKEN=$(kubectl -n media exec deploy/plex -c app -- sh -c "grep -o 'PlexOnlineToken=\"[^\"]*\"' '/config/Library/Application Support/Plex Media Server/Preferences.xml' | cut -d'\"' -f2")
curl -s "http://172.16.0.128:32400/library/sections?X-Plex-Token=$TOKEN" | grep -oE 'title="[^"]+"' | head
kubectl -n network logs deploy/cloudflare-dns --since=10m | grep -iE 'echo|Desired change|error' | head -8
```
Expected: `plex via tunnel http=200`, `.128 http=200`, flux-webhook 404/405 (reachable); the library sections list the existing libraries (no rescan needed); external-dns logs show `DELETE echo.kichi.live` (canary gone) and no errors. Cloudflare GET on the tunnel connections → 1+ connector from the new cluster. Calvin: open Plex → Settings → Remote Access → set manual port 32400 → expect "Fully accessible outside your network" (uses the 32400 forward); play something remotely if possible.

- [ ] **Step 3: VolSync first sync, staging cleanup**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl -n media patch replicationsource plex --type merge -p '{"spec":{"trigger":{"manual":"initial"}}}'
until [ "$(kubectl -n media get replicationsource plex -o jsonpath='{.status.lastManualSync}')" = "initial" ]; do sleep 20; done
kubectl -n media get replicationsource plex -o jsonpath='{.status.latestMoverStatus.result} {.status.lastSyncTime}{"\n"}'
kubectl run stageclean --rm -i --restart=Never --image=busybox:1.36 --overrides='{"spec":{"securityContext":{"runAsUser":1000,"fsGroup":1000},"containers":[{"name":"c","image":"busybox:1.36","command":["sh","-c","rm -rf /nfs/_migration && echo staging removed"],"volumeMounts":[{"name":"nfs","mountPath":"/nfs"}]}],"volumes":[{"name":"nfs","nfs":{"server":"kl-san-1.localdomain","path":"/volume1/data"}}]}}' 2>&1 | grep -vE '^pod |warning|If you|recorded'
```
Expected: `Successful`; R2 usage check afterwards (1.4 GB Plex + ~0.6 GB others + CNPG — well under 10 GB, but note it in the log).

### Task 6: Close out

- [ ] Execution log on `v2` and `main`; memory `migration-progress` (phase 6 done; next = step 7 DNS); rollback note: revert `PR_V2` then `PR_MAIN` (tunnel reconnects to old within ~1 min; Plex restored from the tar on the old side as in phase 4).

---

## Execution log

- PR_V2 / PR_MAIN: _(…)_
- Cloudflare `plex-direct` record id / UniFi DDNS + forward ids: _(Task 2)_
- Plex outage: old stopped _(…)_ → new up _(…)_
- Tar size / entries; libraries listed after restore: _(…)_
- Date completed: _(…)_
