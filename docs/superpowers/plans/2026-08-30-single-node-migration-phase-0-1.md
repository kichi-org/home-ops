# Single-node migration — Phase 0–1 implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the new single-node Talos cluster (`talos-11`, VM 811 on kl-vhost-2) from cluster-template on branch `v2`, with the non-singleton foundation running and both backup layers proven by a restore — ready for the per-app cutovers of phase 2+.

**Architecture:** Orphan branch `v2` in `kichi-org/home-ops` holds the rendered cluster-template output plus ported apps; the old cluster keeps `main`. The new cluster uses disjoint IPs on VLAN 100 (node `.111`, VIP `.20`, gateways `.31/.32/.33`, pool `.30–.99`) so both clusters coexist. Singletons (cloudflare-tunnel, cloudflare-dns, technitium, ARC, apps) are **not** enabled in this phase.

**Tech Stack:** Proxmox 9 (`qm`), Talos Image Factory, cluster-template `b5f9619` (topf, just, uv, makejinja, sops/age, helmfile), Flux Operator, Cilium, envoy-gateway, cert-manager, ESO + 1Password Connect, OpenEBS LocalPV-hostpath, CloudNativePG + Barman Cloud plugin, VolSync/restic, kube-prometheus-stack, grafana-operator, Cloudflare R2.

**Spec:** `docs/superpowers/specs/2026-08-30-single-node-migration-design.md` (sections 3–7, steps 0–1 of section 8).

## Global constraints

- Node `172.16.0.111`, Kube API VIP `172.16.0.20`, envoy-internal `172.16.0.31`, k8s-gateway `172.16.0.32`, envoy-external `172.16.0.33`, LB-IPAM pool `172.16.0.30–172.16.0.99`. Never use `.10–.13`, `.101–.103`, `.122`, `.123`, `.128`, `.200` in this phase (owned by the old cluster).
- VM 811 `talos-11`: 8 vCPU `cpu: host`, 24576 MiB, `balloon 0`, `scsi0` 50 G install, `scsi1` 200 G data, both **without** `backup=0`, on `kingston-nvme`, `net0 virtio,bridge=vmbr0,tag=100`.
- Talos schematic = `siderolabs/qemu-guest-agent` only. Talos/Kubernetes versions = whatever the template pins at `b5f9619` (Renovate keeps them current afterwards).
- Domain `kichi.live`; repo `https://github.com/kichi-org/home-ops.git`, branch `v2`, webhook provider `github`.
- Secrets: app secrets only via ESO/1Password (vault `Kubernetes`); SOPS/age only for the template's bootstrap secrets, re-using the existing `age.key` (recipient `age1qeu5ft3kqvvwz2pkdmmxw7vsrr6gydegmm7km25rqvqexkhga9gq228h6z`).
- StorageClass `openebs-hostpath` (default), basePath `/var/mnt/data/openebs`. Prometheus excluded from backups.
- R2 bucket `kichi`, endpoint `https://932a695a6a0d664160ac575fa0e4fda8.r2.cloudflarestorage.com`, prefixes `cnpg/` and `volsync/<app>/`.
- Nothing in this plan touches the old cluster, `main`'s `kubernetes/` tree, Cloudflare DNS, or the tunnel. Every Proxmox/UniFi/Cloudflare mutation is proposed to Calvin before it runs.
- Work happens in a second checkout `~/repo/kichi-org/home-ops-v2` (git worktree on branch `v2`), so `~/repo/kichi-org/home-ops` stays on `main` for the old cluster.

---

## File map

Branch `v2` (worktree `~/repo/kichi-org/home-ops-v2`):

| Path | Responsibility |
|---|---|
| `cluster.toml` | single source for template rendering (network, nodes, gateways, repo, dns, ingress, talos) |
| `template/config/talos/all/51-nfs.yaml.j2` | carried `/etc/nfsmount.conf` + containerd CRI tweak |
| `template/config/talos/all/52-talos-api-access.yaml.j2` | `kubernetesTalosAPIAccess` for `system-upgrade`, `actions-runner-system` |
| `template/config/talos/all/70-data-volume.yaml.j2` | `UserVolumeConfig data` on `/dev/sdb` + kubelet bind mount |
| `talos/`, `bootstrap/`, `kubernetes/`, `.sops.yaml` | rendered by `just configure` (committed) |
| `kubernetes/apps/external-secrets/{external-secrets,onepassword}` | ported from `main` |
| `kubernetes/apps/openebs-system/openebs` | new: OpenEBS LocalPV-hostpath |
| `kubernetes/apps/database/cloudnative-pg/{app,plugin,cluster}` | ported operator + new Barman Cloud plugin + single-instance cluster with R2 backups |
| `kubernetes/apps/volsync-system/volsync` | new: VolSync operator |
| `kubernetes/components/volsync` | new Flux component: per-app `ReplicationSource` + restic `ExternalSecret` (uses `${APP}`) |
| `kubernetes/components/alerts` | ported Flux → Alertmanager provider/alert |
| `kubernetes/apps/observability/kube-prometheus-stack` | ported, storageClass swap, retention 7d/15GB, backup PrometheusRules added |
| `kubernetes/apps/observability/grafana/{app,instance}` | ported, storageClass swap, VolSync component |
| `docs/` | copied from `main` so the design/plan survive the eventual `v2 → main` |

Branch `main` (this checkout): only `.renovaterc.json5` (`baseBranches`) and `docs/` change.

---

## Phase 0 — preparation

### Task 0.1: Refresh the template clone and pin the commit

**Files:** none in home-ops; `/Users/calvin/repo/cluster-template`.

- [ ] **Step 1: Pull and record the commit**

```bash
cd /Users/calvin/repo/cluster-template && git pull --ff-only && git log -1 --format='%h %cs %s'
```
Expected: a hash ≥ `b5f9619`. Write the hash into the "Template commit" line at the end of this plan (section "Execution log").

- [ ] **Step 2: If the hash moved, diff the parts this plan depends on**

```bash
cd /Users/calvin/repo/cluster-template && git diff b5f9619..HEAD --stat -- cluster.sample.toml template/scripts/validate.py template/config/talos template/config/bootstrap justfile template/mod.just
```
Expected: ideally empty. If not empty, read the diff and adjust Task 0.4's `cluster.toml` keys / patch filenames before continuing.

- [ ] **Step 3: Install the toolchain the template expects**

```bash
cd /Users/calvin/repo/cluster-template && mise trust && mise install && mise ls --current | grep -E 'just|uv|topf|talosctl|kubectl|helmfile|sops|age|flux|kubeconform|gum|yq'
```
Expected: every tool listed with a version. (`mise` is already installed from the old setup; if `mise` itself is missing: `brew install mise`.)

### Task 0.2: Build the Talos schematic and stage the ISO on `san-iso`

**Files:** none in git. Output: schematic ID, ISO on `san-iso`.

**Produces:** `SCHEMATIC_ID` (64-hex) and ISO volume `san-iso:iso/talos-<ver>-qemu-metal-amd64.iso` used by Tasks 0.3 and 0.4.

- [ ] **Step 1: Read the Talos version the template pins**

```bash
grep -rn 'talosVersion\|talos_version\|siderolabs/installer' /Users/calvin/repo/cluster-template/template/config/talos/topf.yaml.j2 /Users/calvin/repo/cluster-template/template/scripts/validate.py | head
```
Expected: one pinned `vX.Y.Z` (call it `TALOS_VER`).

- [ ] **Step 2: Create the schematic (qemu-guest-agent only)**

```bash
cat > /private/tmp/claude-501/-Users-calvin-repo-kichi-org-home-ops/ade757b7-b18a-4827-b665-0f8b26076faf/scratchpad/schematic.yaml <<'EOF'
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/qemu-guest-agent
EOF
curl -sS -X POST --data-binary @/private/tmp/claude-501/-Users-calvin-repo-kichi-org-home-ops/ade757b7-b18a-4827-b665-0f8b26076faf/scratchpad/schematic.yaml https://factory.talos.dev/schematics
```
Expected: `{"id":"<64-hex>"}`. Record as `SCHEMATIC_ID`. (Deterministic — re-posting the same YAML returns the same ID.)

- [ ] **Step 3: Verify the ISO URL resolves**

```bash
curl -sSI "https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VER}/metal-amd64.iso" | head -1
```
Expected: `HTTP/2 200`.

- [ ] **Step 4: Download the ISO onto `san-iso` from kl-vhost-2** (propose to Calvin first; runs on the host via SSH, or via Proxmox MCP `download_iso` on `proxmox-vhost2`)

```bash
ssh root@192.168.85.251 "cd /mnt/pve/san-iso/template/iso && curl -sSLo talos-${TALOS_VER}-qemu-metal-amd64.iso https://factory.talos.dev/image/${SCHEMATIC_ID}/${TALOS_VER}/metal-amd64.iso && ls -l talos-${TALOS_VER}-qemu-metal-amd64.iso"
```
Expected: file ≈ 100–300 MiB. Do **not** overwrite the existing `metal-amd64.iso` (unknown provenance).

- [ ] **Step 5: Confirm Proxmox sees it**

Proxmox MCP `proxmox-vhost2` → `list_isos` (storage `san-iso`).
Expected: `san-iso:iso/talos-<ver>-qemu-metal-amd64.iso` listed.

### Task 0.3: Create VM 811 on kl-vhost-2

**Files:** none in git. Output: VM 811 and its MAC (`VM_MAC`, lowercase) for Task 0.4.

**Produces:** VM 811 booted into Talos maintenance mode with a DHCP lease in `.220–.254`; `VM_MAC`.

- [ ] **Step 1: Propose and run the create command** (Calvin runs it over SSH — the MCP tokens are read-only and `create_vm` does not expose every `qm` flag; a scoped RW token for start/stop/snapshot/backup is optional, see spec follow-ups)

```bash
ssh root@192.168.85.251 qm create 811 \
  --name talos-11 --ostype l26 \
  --cores 8 --sockets 1 --cpu host --numa 0 \
  --memory 24576 --balloon 0 \
  --scsihw virtio-scsi-single --agent 1 \
  --scsi0 kingston-nvme:50,discard=on,iothread=1,ssd=1 \
  --scsi1 kingston-nvme:200,discard=on,iothread=1,ssd=1 \
  --ide2 san-iso:iso/talos-${TALOS_VER}-qemu-metal-amd64.iso,media=cdrom \
  --boot 'order=ide2;scsi0;net0' \
  --net0 virtio,bridge=vmbr0,tag=100 \
  --onboot 1 --startup order=2,up=30
```
Expected: exit 0. Then read back: Proxmox MCP `get_vm_config` (node `kl-vhost-2`, vmid `811`) → both `scsi0`/`scsi1` **without** `backup=0`, `net0` shows the generated MAC → record lowercase as `VM_MAC`.

- [ ] **Step 2: Boot into maintenance mode**

Proxmox MCP `start_vm` (`kl-vhost-2`, `811`) after Calvin's go. Then find the lease:
UniFi MCP `unifi_execute` → `unifi_list_clients` filtered to `172.16.0.22x–25x`; or
```bash
talosctl get links -n <lease-ip> --insecure | head
talosctl get disks -n <lease-ip> --insecure
```
Expected: `sda` 50 GB, `sdb` 200 GB, link with the VM's MAC. Record `MAINT_IP`.

---

## Phase 1 — foundation
### Task 0.4: Create branch `v2` from the template and render it

**Files:**
- Create worktree `~/repo/kichi-org/home-ops-v2` on orphan branch `v2`
- Create: `cluster.toml`, `template/config/talos/all/{51-nfs,52-talos-api-access,70-data-volume}.yaml.j2`
- Copy in (gitignored): `age.key`, `cloudflare-tunnel.json`, `flux-webhook-token.txt`
- Rendered: `talos/`, `bootstrap/`, `kubernetes/`, `.sops.yaml`

**Produces:** branch `v2` pushed with a bootable, validated render; the network apps `cloudflare-tunnel`, `cloudflare-dns` and `default/echo` removed from the Flux tree.

- [ ] **Step 1: Create the orphan worktree and seed it with the template**

```bash
cd /Users/calvin/repo/kichi-org/home-ops
git worktree add --orphan -b v2 ../home-ops-v2
rsync -a --exclude .git --exclude .github/template-tests /Users/calvin/repo/cluster-template/ ../home-ops-v2/
cd ../home-ops-v2 && ls
```
Expected: `LICENSE README.md cluster.sample.toml justfile makejinja.toml pyproject.toml template uv.lock` plus dotfiles (`.mise`, `.github`, `.gitignore`, `.sops.yaml` is generated later).

- [ ] **Step 2: Bring over the secret inputs (all gitignored by the template's `.gitignore`)**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2
cp ../home-ops/age.key ./age.key
cp ../home-ops/cloudflare-tunnel.json ./cloudflare-tunnel.json
export SOPS_AGE_KEY_FILE=$PWD/age.key
sops decrypt ../home-ops/kubernetes/apps/flux-system/flux-instance/app/secret.sops.yaml | yq -r '.stringData.token' > flux-webhook-token.txt
sops decrypt ../home-ops/kubernetes/apps/network/cloudflare-dns/app/secret.sops.yaml | yq -r '.stringData["api-token"]'   # → CF_DNS_TOKEN, paste into cluster.toml [dns].token
git check-ignore -v age.key cloudflare-tunnel.json flux-webhook-token.txt
```
Expected: all three paths reported as ignored. The Cloudflare token needs `Zone:DNS:Edit` + `Account:Cloudflare Tunnel:Read`; if the old token lacks the Tunnel scope (template's external-dns reads tunnel info), create a new token in the Cloudflare dashboard with both scopes.

- [ ] **Step 3: `just init` (creates `cluster.toml`, `deploy.key`, keeps existing files)**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2 && mise trust && mise install && just init && ls -la cluster.toml deploy.key deploy.key.pub flux-webhook-token.txt age.key
```
Expected: all five files exist; `age.key` unchanged (`diff age.key ../home-ops/age.key` is empty).

- [ ] **Step 4: Write `cluster.toml`**

```toml
[network]
node_cidr = "172.16.0.0/24"
dns_servers = ["172.16.0.200", "192.168.85.1"]
ntp_servers = ["162.159.200.1", "162.159.200.123"]
default_gateway = "172.16.0.1"

[kubernetes]
pod_cidr = "10.42.0.0/16"
svc_cidr = "10.43.0.0/16"

[kubernetes.api]
addr = "172.16.0.20"

[gateways]
internal = "172.16.0.31"
dns = "172.16.0.32"
external = "172.16.0.33"

[repository]
url = "https://github.com/kichi-org/home-ops.git"
branch = "v2"
webhook_provider = "github"

[domain]
name = "kichi.live"

[dns]
provider = "cloudflare"
token = "<CF_DNS_TOKEN from Step 2>"

[ingress]
mode = "cloudflare-tunnel"

[cilium]
loadbalancer_mode = "dsr"

[talos]
schematic_id = "<SCHEMATIC_ID from Task 0.2>"

[[nodes]]
name = "talos-11"
address = "172.16.0.111"
controller = true
disk = "/dev/sda"
mac_addr = "<VM_MAC from Task 0.3>"
```

- [ ] **Step 5: Add the carried Talos patches under `template/config/talos/all/`**

`template/config/talos/all/51-nfs.yaml.j2`:
```yaml
machine:
  files:
    - op: create
      path: /etc/cri/conf.d/20-customization.part
      content: |
        [plugins."io.containerd.cri.v1.images"]
          discard_unpacked_layers = false
        [plugins."io.containerd.cri.v1.runtime"]
          device_ownership_from_security_context = true
    - op: overwrite
      path: /etc/nfsmount.conf
      permissions: 0o644
      content: |
        [ NFSMount_Global_Options ]
        nfsvers=4.1
        hard=True
        noatime=True
        nconnect=16
```

`template/config/talos/all/52-talos-api-access.yaml.j2`:
```yaml
machine:
  features:
    kubernetesTalosAPIAccess:
      enabled: true
      allowedRoles:
        - os:admin
      allowedKubernetesNamespaces:
        - system-upgrade
        - actions-runner-system
```

`template/config/talos/all/70-data-volume.yaml.j2`:
```yaml
machine:
  kubelet:
    extraMounts:
      - destination: /var/mnt/data
        type: bind
        source: /var/mnt/data
        options:
          - bind
          - rshared
          - rw
---
apiVersion: v1alpha1
kind: UserVolumeConfig
name: data
provisioning:
  diskSelector:
    match: disk.dev_path == '/dev/sdb'
  grow: true
  minSize: 190GiB
filesystem:
  type: xfs
```

Then check the template doesn't already carry `kubernetesTalosAPIAccess` or the old sysctls (avoid duplicate keys):
```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2 && grep -rn 'kubernetesTalosAPIAccess\|nfsmount' template/config/talos/ ; grep -c . template/config/talos/all/40-sysctls.yaml.j2
```
Expected: only your new files match. If `kubernetesTalosAPIAccess` already exists in a template file, delete `52-…` and add the two namespaces there instead.

- [ ] **Step 6: Render, encrypt, validate**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2 && just configure 2>&1 | tail -20
```
Expected: ends with kubeconform + `topf render` success, no `undefined` Jinja errors. Then:
```bash
grep -n 'sops:' talos/secrets.sops.yaml | head -1 && ls talos/ bootstrap/ kubernetes/apps/
```
Expected: `secrets.sops.yaml` encrypted; namespaces `cert-manager default flux-system kube-system network`.

- [ ] **Step 7: Remove the singletons and echo from the rendered tree**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2
rm -r kubernetes/apps/network/cloudflare-tunnel kubernetes/apps/network/cloudflare-dns kubernetes/apps/default
yq -i '.resources -= ["./cloudflare-tunnel/ks.yaml", "./cloudflare-dns/ks.yaml"]' kubernetes/apps/network/kustomization.yaml
cat kubernetes/apps/network/kustomization.yaml
```
Expected: resources = `namespace.yaml`, `envoy-gateway/ks.yaml`, `k8s-gateway/ks.yaml` only. Also remove `default` from `bootstrap/helmfile/apps.yaml` only if it references echo (`grep -n echo bootstrap/helmfile/*.yaml` → expected none). Keep the `flux-webhook` HTTPRoute (harmless until the tunnel moves).

Note: `just configure` re-renders `kubernetes/` and would restore these files. From here on, **never re-run `just configure`**; use `just template render`-free workflows: `just talos render` for Talos only, and edit `kubernetes/` by hand. (Task 1.9 archives the template with `just template tidy`.) If a Talos-only change is ever needed before that, edit `talos/` directly or run `just template render` and then `git checkout -- kubernetes && git clean -fd kubernetes` to drop the resurrected apps.

- [ ] **Step 8: Copy the docs and commit/push `v2`**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2
mkdir -p docs && cp -R ../home-ops/docs/. docs/
git add -A && git status --short | grep -E 'age.key|cloudflare-tunnel.json|flux-webhook-token|deploy.key$|talos/rendered' ; echo "exit=$?"
```
Expected: `exit=1` (no secret file staged). Then:
```bash
git commit -m "chore: initial commit :rocket:

Rendered from onedr0p/cluster-template <hash> for the single-node rebuild
(docs/superpowers/specs/2026-08-30-single-node-migration-design.md)." && git push -u origin v2
```
Expected: branch `v2` on GitHub. `flate`/kubeconform CI from the template runs on `v2` — check it is green.
### Task 0.5: Point Renovate at `v2` (change on `main`)

**Files:** Modify `.renovaterc.json5` on `main`.

- [ ] **Step 1: Add `baseBranches`**

In `/Users/calvin/repo/kichi-org/home-ops/.renovaterc.json5`, add at top level:
```json5
  baseBranches: ["v2"],
```
Verify:
```bash
cd /Users/calvin/repo/kichi-org/home-ops && npx --yes renovate-config-validator .renovaterc.json5
```
Expected: `Config validated successfully`.

- [ ] **Step 2: Commit on a branch, open PR, Calvin merges**

```bash
git checkout -b chore/renovate-v2 && git add .renovaterc.json5 && git commit -m "chore(renovate): target branch v2 during the single-node rebuild" && git push -u origin chore/renovate-v2
```
Open the PR (GitHub MCP `create_pull_request`, base `main`). After merge, the hourly run will open PRs against `v2` only; the 15 open PRs against `main` can be closed (old cluster frozen).

- [ ] **Step 3: Verify after the next hourly run**

GitHub MCP `list_pull_requests` (repo `kichi-org/home-ops`, state open) → new PRs have `base: v2`.

### Task 1.1: Bootstrap Talos and Flux

**Files:** rendered `talos/`, `bootstrap/`; outputs `talos/talosconfig`, `kubeconfig` (gitignored).

**Produces:** node `talos-11` Ready at `172.16.0.111`, API on `https://172.16.0.20:6443`, Flux reconciling `v2`.

- [ ] **Step 1: Apply and bootstrap**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2 && export SOPS_AGE_KEY_FILE=$PWD/age.key
just bootstrap talos 2>&1 | tail -30
```
(`topf apply --auto-bootstrap` discovers the node by MAC in maintenance mode; if it cannot, run `just talos apply-node talos-11` with `--nodes` pointed at `MAINT_IP` per `talos/README.md`.)
Expected: node installs to `/dev/sda`, reboots, etcd bootstraps; `talos/talosconfig` and `./kubeconfig` written.

- [ ] **Step 2: Verify node and data volume**

```bash
export TALOSCONFIG=$PWD/talos/talosconfig KUBECONFIG=$PWD/kubeconfig
talosctl -n 172.16.0.111 get volumestatus | grep -E 'u-data|EPHEMERAL'
talosctl -n 172.16.0.111 get mountstatus | grep /var/mnt/data
talosctl -n 172.16.0.111 read /etc/nfsmount.conf
kubectl get nodes -o wide
```
Expected: `u-data` ready ~190 GiB xfs mounted at `/var/mnt/data`; nfsmount.conf shows `nconnect=16`; node `NotReady` (no CNI yet) at `172.16.0.111`.

- [ ] **Step 3: Bootstrap apps (Cilium → CoreDNS → cert-manager → flux-operator → flux-instance, plus CRDs)**

```bash
just bootstrap apps 2>&1 | tail -30
kubectl get nodes && kubectl -n flux-system get fluxinstance,gitrepository,kustomization
```
Expected: node `Ready`; `GitRepository flux-system` revision `v2@sha1:…`; Kustomizations `flux-system`, `cluster-apps`, `cilium`, `coredns`, `cert-manager`, `flux-operator`, `flux-instance`, `envoy-gateway`, `k8s-gateway`, `metrics-server`, `reloader` all `Ready=True` within ~10 min.

- [ ] **Step 4: Verify addressing, TLS and DNS on the transition IPs**

```bash
kubectl -n network get svc -o wide | grep -E 'envoy|k8s-gateway'
kubectl -n network get certificate
dig +short @172.16.0.32 flux-webhook.kichi.live
curl -skI --resolve flux-webhook.kichi.live:443:172.16.0.33 https://flux-webhook.kichi.live/ | head -1
```
Expected: `envoy-internal 172.16.0.31`, `envoy-external 172.16.0.33`, `k8s-gateway 172.16.0.32`; Certificate `READY=True` (Let's Encrypt wildcard via DNS-01 — coexists with the old cluster's); dig returns `172.16.0.33`; curl gets an HTTP status (404 is fine).

Rate-limit guard: both clusters request the identical `kichi.live` + `*.kichi.live` name set, and Let's Encrypt allows only 5 duplicate certificates per week. Check the profile:
```bash
kubectl get clusterissuer -o yaml | grep -n 'profile:'
kubectl -n network get certificate -o jsonpath='{.items[0].spec.duration}{"\n"}'
```
If the issuer uses `profile: shortlived` (160 h certs), remove the `profile` line from `kubernetes/apps/cert-manager/cert-manager/app/` (ClusterIssuer) and any `duration:`/`renewBefore:` on the Certificate so the new cluster uses 90-day certs until the old cluster is retired; commit as `fix(cert-manager): classic profile during dual-cluster transition`.

- [ ] **Step 5: Commit any bootstrap-generated tracked changes**

```bash
git status --short && git add -A && git commit -m "chore: post-bootstrap" || true
```
Expected: usually nothing to commit.

### Task 1.2: External Secrets + 1Password Connect

**Files:**
- Create: `kubernetes/apps/external-secrets/{namespace.yaml,kustomization.yaml}`, `external-secrets/{ks.yaml,app/*}`, `onepassword/{ks.yaml,app/*}` — copied from `main`
- Manual: Secret `onepassword-secret` in `external-secrets` (seed, not in git)

**Produces:** `ClusterSecretStore/onepassword` Ready; every later `ExternalSecret` with `secretStoreRef: {kind: ClusterSecretStore, name: onepassword}` works.

- [ ] **Step 1: Port the namespace from `main`**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2
cp -R ../home-ops/kubernetes/apps/external-secrets kubernetes/apps/external-secrets
sed -i '' 's/replicaCount: 2/replicaCount: 1/' kubernetes/apps/external-secrets/external-secrets/app/helmrelease.yaml
grep -rn 'components:' -A1 kubernetes/apps/external-secrets/kustomization.yaml
```
Expected: `components: [../../components/alerts]` — that component does not exist yet on `v2`; change it to `../../components/sops` for now (Task 1.6 adds `alerts` and switches it back). Remove the PDB file if present (`grep -l PodDisruptionBudget -r kubernetes/apps/external-secrets` → delete + drop from `kustomization.yaml`).

- [ ] **Step 2: Create the seed secret from 1Password (before pushing, so ESO doesn't crash-loop)**

```bash
op read "op://Kubernetes/1password/1password-credentials.json" > /private/tmp/claude-501/-Users-calvin-repo-kichi-org-home-ops/ade757b7-b18a-4827-b665-0f8b26076faf/scratchpad/1password-credentials.json
kubectl create namespace external-secrets --dry-run=client -o yaml | kubectl apply -f -
kubectl -n external-secrets create secret generic onepassword-secret \
  --from-file=1password-credentials.json=/private/tmp/claude-501/-Users-calvin-repo-kichi-org-home-ops/ade757b7-b18a-4827-b665-0f8b26076faf/scratchpad/1password-credentials.json \
  --from-literal=token="$(op read 'op://Kubernetes/1password/token')"
rm /private/tmp/claude-501/-Users-calvin-repo-kichi-org-home-ops/ade757b7-b18a-4827-b665-0f8b26076faf/scratchpad/1password-credentials.json
kubectl -n external-secrets get secret onepassword-secret -o jsonpath='{.data}' | jq 'keys'
```
Expected: `["1password-credentials.json","token"]`. (If the item's field names differ, `op item get 1password --vault Kubernetes` and adapt — the ExternalSecret in `onepassword/app` reads exactly these two keys.)

- [ ] **Step 3: Commit, push, reconcile, verify**

```bash
git add kubernetes/apps/external-secrets && git commit -m "feat(external-secrets): port ESO + 1Password Connect" && git push
just kube reconcile
kubectl -n external-secrets get pods && kubectl get clustersecretstore onepassword
```
Expected: `external-secrets`, `external-secrets-webhook`, `external-secrets-cert-controller`, `onepassword` pods Running; `ClusterSecretStore onepassword STATUS=Valid READY=True`.

- [ ] **Step 4: Prove a round-trip with a throwaway ExternalSecret**

```bash
kubectl apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: {name: probe, namespace: external-secrets}
spec:
  secretStoreRef: {kind: ClusterSecretStore, name: onepassword}
  target: {name: probe}
  dataFrom: [{extract: {key: pushover}}]
EOF
sleep 10; kubectl -n external-secrets get externalsecret probe; kubectl -n external-secrets delete externalsecret probe
```
Expected: `STATUS=SecretSynced READY=True`.

### Task 1.3: OpenEBS LocalPV-hostpath (default StorageClass)

**Files:**
- Create: `kubernetes/apps/openebs-system/namespace.yaml`, `kustomization.yaml`, `openebs/ks.yaml`, `openebs/app/{kustomization,helmrepository,helmrelease}.yaml`

**Produces:** StorageClass `openebs-hostpath` (default, `WaitForFirstConsumer`, `Delete`) backed by `/var/mnt/data/openebs`.

- [ ] **Step 1: Look up and pin the chart version**

```bash
helm repo add openebs https://openebs.github.io/openebs >/dev/null && helm repo update openebs >/dev/null && helm search repo openebs/openebs --versions | head -3
```
Expected: latest `openebs/openebs` `4.x.y` — use it as `<OPENEBS_VER>` below.

- [ ] **Step 2: Write the manifests**

`kubernetes/apps/openebs-system/namespace.yaml`:
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: openebs-system
  labels:
    pod-security.kubernetes.io/enforce: privileged
```
`kubernetes/apps/openebs-system/kustomization.yaml`:
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./namespace.yaml
  - ./openebs/ks.yaml
```
`kubernetes/apps/openebs-system/openebs/ks.yaml`:
```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: openebs
spec:
  interval: 1h
  path: ./kubernetes/apps/openebs-system/openebs/app
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  targetNamespace: openebs-system
  wait: true
  healthChecks:
    - apiVersion: storage.k8s.io/v1
      kind: StorageClass
      name: openebs-hostpath
```
`kubernetes/apps/openebs-system/openebs/app/kustomization.yaml`:
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./helmrepository.yaml
  - ./helmrelease.yaml
```
`kubernetes/apps/openebs-system/openebs/app/helmrepository.yaml`:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: openebs
spec:
  interval: 2h
  url: https://openebs.github.io/openebs
```
`kubernetes/apps/openebs-system/openebs/app/helmrelease.yaml`:
```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: openebs
spec:
  interval: 1h
  chart:
    spec:
      chart: openebs
      version: <OPENEBS_VER>
      sourceRef:
        kind: HelmRepository
        name: openebs
  values:
    localpv-provisioner:
      localpv:
        image:
          registry: quay.io/
      hostpathClass:
        enabled: true
        name: openebs-hostpath
        isDefaultClass: true
        basePath: /var/mnt/data/openebs
        reclaimPolicy: Delete
    engines:
      local:
        lvm: {enabled: false}
        zfs: {enabled: false}
      replicated:
        mayastor: {enabled: false}
    loki: {enabled: false}
    alloy: {enabled: false}
    minio: {enabled: false}
    openebs-crds:
      csi:
        volumeSnapshots:
          enabled: false
```
Validate the values keys against the chart before committing:
```bash
helm show values openebs/openebs --version <OPENEBS_VER> | grep -nE '^(engines|loki|alloy|minio|localpv-provisioner|openebs-crds):' 
```
Expected: every top-level key you used is present (adapt names if the chart renamed them; the `volumeSnapshots` CRD must stay off because VolSync `Direct` does not need it and the old cluster's Longhorn snapshot CRDs are not here).

- [ ] **Step 3: Commit, push, verify**

```bash
git add kubernetes/apps/openebs-system && git commit -m "feat(openebs): localpv-hostpath on /var/mnt/data" && git push && just kube reconcile
kubectl get storageclass && kubectl -n openebs-system get pods
```
Expected: `openebs-hostpath (default)` with `openebs.io/local` provisioner, `WaitForFirstConsumer`; provisioner pod Running.

- [ ] **Step 4: PVC smoke test**

```bash
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: probe, namespace: default}
spec: {accessModes: [ReadWriteOnce], resources: {requests: {storage: 1Gi}}}
---
apiVersion: v1
kind: Pod
metadata: {name: probe, namespace: default}
spec:
  containers: [{name: w, image: busybox:1.36, command: [sh, -c, "echo ok > /d/ok && sleep 3600"], volumeMounts: [{name: d, mountPath: /d}]}]
  volumes: [{name: d, persistentVolumeClaim: {claimName: probe}}]
EOF
sleep 20; kubectl -n default get pvc probe; talosctl -n 172.16.0.111 ls /var/mnt/data/openebs
kubectl -n default delete pod probe pvc probe
```
Expected: PVC `Bound`; a `pvc-…` directory under `/var/mnt/data/openebs`; gone after delete.

### Task 1.4: CloudNativePG with Barman Cloud plugin → R2

**Files:**
- Create: `kubernetes/apps/database/{namespace.yaml,kustomization.yaml}`; `cloudnative-pg/ks.yaml`; `cloudnative-pg/app/*` (ported operator); `cloudnative-pg/plugin/{kustomization,ocirepository,helmrelease}.yaml`; `cloudnative-pg/cluster/{kustomization,externalsecret,objectstore,cluster,scheduledbackup,prometheusrule}.yaml`

**Interfaces:**
- Consumes: `ClusterSecretStore/onepassword` (Task 1.2); StorageClass `openebs-hostpath` (Task 1.3); 1Password items `cloudnative-pg` (superuser creds) and an R2 item.
- Produces: Service `postgres-rw.database.svc.cluster.local:5432`, `ObjectStore/r2`, one nightly base backup + continuous WAL in `s3://kichi/cnpg/postgres/`. TeslaMate/Grafana (phase 2) use `postgres-rw`.

- [ ] **Step 1: Find the R2 credential item in 1Password**

```bash
op item list --vault Kubernetes --format json | jq -r '.[].title' | grep -iE 'r2|s3|longhorn|cloudflare'
op item get <item> --vault Kubernetes --fields label=AWS_ACCESS_KEY_ID,label=AWS_SECRET_ACCESS_KEY --format json | jq -r '.[].label'
```
Expected: one item with S3-style keys (the old `longhorn-s3-secret` source). Record its title as `R2_ITEM` and the two field labels. If the field labels are not `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`, map them in the ExternalSecret `data:` below instead of `dataFrom`.

- [ ] **Step 2: Port the operator, pin the plugin chart**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2
mkdir -p kubernetes/apps/database && cp ../home-ops/kubernetes/apps/database/namespace.yaml kubernetes/apps/database/
cp -R ../home-ops/kubernetes/apps/database/cloudnative-pg kubernetes/apps/database/cloudnative-pg
rm -rf kubernetes/apps/database/cloudnative-pg/cluster/*   # rewritten below
helm show chart oci://ghcr.io/cloudnative-pg/charts/plugin-barman-cloud 2>/dev/null | grep -E '^version:' || echo "chart not at that OCI path — check https://github.com/cloudnative-pg/charts"
```
Expected: a version (`<PLUGIN_VER>`). Note the old `cloudnative-pg-app/ks.yaml` has an empty `dependsOn:` key — delete that key.

- [ ] **Step 3: Write the namespace/ks files**

`kubernetes/apps/database/kustomization.yaml`:
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
components:
  - ../../components/sops
resources:
  - ./namespace.yaml
  - ./cloudnative-pg/ks.yaml
```
`kubernetes/apps/database/cloudnative-pg/ks.yaml` (three Kustomizations):
```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cloudnative-pg
spec:
  interval: 1h
  path: ./kubernetes/apps/database/cloudnative-pg/app
  prune: true
  sourceRef: {kind: GitRepository, name: flux-system, namespace: flux-system}
  targetNamespace: database
  wait: true
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cloudnative-pg-plugin
spec:
  dependsOn:
    - name: cloudnative-pg
    - name: cert-manager
      namespace: cert-manager
  interval: 1h
  path: ./kubernetes/apps/database/cloudnative-pg/plugin
  prune: true
  sourceRef: {kind: GitRepository, name: flux-system, namespace: flux-system}
  targetNamespace: database
  wait: true
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cloudnative-pg-cluster
spec:
  dependsOn:
    - name: cloudnative-pg-plugin
    - name: openebs
      namespace: openebs-system
    - name: onepassword
      namespace: external-secrets
  interval: 1h
  path: ./kubernetes/apps/database/cloudnative-pg/cluster
  prune: true
  sourceRef: {kind: GitRepository, name: flux-system, namespace: flux-system}
  targetNamespace: database
  wait: false
  healthChecks:
    - apiVersion: postgresql.cnpg.io/v1
      kind: Cluster
      name: postgres
      namespace: database
```

- [ ] **Step 4: Plugin HelmRelease**

`kubernetes/apps/database/cloudnative-pg/plugin/kustomization.yaml`:
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./ocirepository.yaml
  - ./helmrelease.yaml
```
`kubernetes/apps/database/cloudnative-pg/plugin/ocirepository.yaml`:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: plugin-barman-cloud
spec:
  interval: 1h
  layerSelector: {mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip, operation: copy}
  ref: {tag: <PLUGIN_VER>}
  url: oci://ghcr.io/cloudnative-pg/charts/plugin-barman-cloud
```
`kubernetes/apps/database/cloudnative-pg/plugin/helmrelease.yaml`:
```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: plugin-barman-cloud
spec:
  interval: 1h
  chartRef: {kind: OCIRepository, name: plugin-barman-cloud}
  values:
    certManager:
      enabled: true
```

- [ ] **Step 5: Cluster, ObjectStore, ScheduledBackup**

`kubernetes/apps/database/cloudnative-pg/cluster/kustomization.yaml`:
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./externalsecret.yaml
  - ./objectstore.yaml
  - ./cluster.yaml
  - ./scheduledbackup.yaml
  - ./prometheusrule.yaml
```
`kubernetes/apps/database/cloudnative-pg/cluster/externalsecret.yaml`:
```yaml
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: cloudnative-pg-secret
spec:
  secretStoreRef: {kind: ClusterSecretStore, name: onepassword}
  target:
    name: cloudnative-pg-secret
    template:
      type: kubernetes.io/basic-auth
      data:
        username: "{{ .POSTGRES_SUPER_USER }}"
        password: "{{ .POSTGRES_SUPER_PASS }}"
  dataFrom:
    - extract: {key: cloudnative-pg}
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: cloudnative-pg-r2-secret
spec:
  secretStoreRef: {kind: ClusterSecretStore, name: onepassword}
  target:
    name: cloudnative-pg-r2-secret
  dataFrom:
    - extract: {key: <R2_ITEM>}
```
(Check the old `cluster/externalsecret.yaml` on `main` for the exact superuser field names and keep those.)

`kubernetes/apps/database/cloudnative-pg/cluster/objectstore.yaml`:
```yaml
---
apiVersion: barmancloud.cnpg.io/v1
kind: ObjectStore
metadata:
  name: r2
spec:
  retentionPolicy: 7d
  configuration:
    destinationPath: s3://kichi/cnpg/
    endpointURL: https://932a695a6a0d664160ac575fa0e4fda8.r2.cloudflarestorage.com
    s3Credentials:
      accessKeyId: {name: cloudnative-pg-r2-secret, key: AWS_ACCESS_KEY_ID}
      secretAccessKey: {name: cloudnative-pg-r2-secret, key: AWS_SECRET_ACCESS_KEY}
    wal:
      compression: gzip
      maxParallel: 2
    data:
      compression: gzip
```
`kubernetes/apps/database/cloudnative-pg/cluster/cluster.yaml`:
```yaml
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: postgres
spec:
  instances: 1
  imageName: <same image as main's cluster.yaml>
  primaryUpdateStrategy: unsupervised
  storage:
    size: 10Gi
    storageClass: openebs-hostpath
  superuserSecret:
    name: cloudnative-pg-secret
  enableSuperuserAccess: true
  postgresql:
    parameters:
      max_connections: "300"
      shared_buffers: 512MB
  monitoring:
    enablePodMonitor: true
  plugins:
    - name: barman-cloud.cloudnative-pg.io
      isWALArchiver: true
      parameters:
        barmanObjectName: r2
```
`kubernetes/apps/database/cloudnative-pg/cluster/scheduledbackup.yaml`:
```yaml
---
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: postgres-daily
spec:
  schedule: "0 0 3 * * *"
  immediate: true
  backupOwnerReference: self
  cluster: {name: postgres}
  method: plugin
  pluginConfiguration:
    name: barman-cloud.cloudnative-pg.io
```
`kubernetes/apps/database/cloudnative-pg/cluster/prometheusrule.yaml`:
```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cloudnative-pg-backups
spec:
  groups:
    - name: cloudnative-pg.backups
      rules:
        - alert: CNPGBackupStale
          expr: time() - max by (cluster) (cnpg_collector_last_available_backup_timestamp) > 36 * 3600
          for: 30m
          labels: {severity: critical}
          annotations:
            summary: "CNPG {{ $labels.cluster }}: last base backup older than 36h"
        - alert: CNPGWALArchiveFailing
          expr: increase(cnpg_collector_pg_wal_archive_status{value="failed"}[30m]) > 0
          for: 15m
          labels: {severity: critical}
          annotations:
            summary: "CNPG {{ $labels.cluster }}: WAL archiving to R2 is failing"
```

- [ ] **Step 6: Commit, push, verify operator + plugin + cluster**

```bash
git add kubernetes/apps/database && git commit -m "feat(cloudnative-pg): single-instance cluster with barman-cloud backups to R2" && git push && just kube reconcile
kubectl -n database get pods,cluster,objectstore
```
Expected: `cloudnative-pg-*` and `barman-cloud-*` pods Running; `Cluster postgres` `Cluster in healthy state`, 1 instance; `ObjectStore r2` present.

- [ ] **Step 7: Verify WAL archiving and the immediate backup**

```bash
kubectl -n database get backup
kubectl -n database exec postgres-1 -c postgres -- sh -c 'psql -U postgres -tAc "select last_archived_wal, last_failed_wal from pg_stat_archiver"'
```
Expected: one `Backup` `completed`; `last_archived_wal` non-empty, `last_failed_wal` empty. Cross-check in R2 (Cloudflare MCP GET on the bucket, or `aws s3 ls --endpoint-url … s3://kichi/cnpg/postgres/`): `base/` and `wals/` prefixes exist.

- [ ] **Step 8: Verify the metric names used by the alerts exist**

```bash
kubectl -n database exec postgres-1 -c postgres -- sh -c 'wget -qO- localhost:9187/metrics' | grep -E '^cnpg_collector_(last_available_backup_timestamp|pg_wal_archive_status)'
```
Expected: both series present. If a name differs, fix `prometheusrule.yaml` now.

### Task 1.5: VolSync operator + reusable backup component

**Files:**
- Create: `kubernetes/apps/volsync-system/{namespace.yaml,kustomization.yaml}`, `volsync/ks.yaml`, `volsync/app/{kustomization,ocirepository,helmrelease}.yaml`
- Create: `kubernetes/components/volsync/{kustomization,externalsecret,replicationsource}.yaml`

**Interfaces:**
- Consumes: `ClusterSecretStore/onepassword`, R2 item `<R2_ITEM>` (Task 1.4 Step 1), a 1Password item `volsync` with field `RESTIC_PASSWORD`.
- Produces: Flux component `../../components/volsync` — any app Kustomization that sets `postBuild.substitute.APP=<name>` and adds the component gets an hourly restic backup of PVC `<name>` to `s3://kichi/volsync/<name>`.

- [ ] **Step 1: Create the restic password item in 1Password (Calvin, once)**

`op item create --vault Kubernetes --category password --title volsync RESTIC_PASSWORD="$(openssl rand -base64 32)"` — record nothing; ESO reads it. Verify: `op item get volsync --vault Kubernetes --fields label=RESTIC_PASSWORD | wc -c` → > 30.

- [ ] **Step 2: Pin the chart and write the operator**

```bash
helm show chart oci://ghcr.io/home-operations/charts-mirror/volsync 2>/dev/null | grep '^version:' || helm show chart oci://ghcr.io/backube/helm-charts/volsync | grep '^version:'
```
Expected: `<VOLSYNC_VER>` and which OCI url worked (`<VOLSYNC_OCI>`).

`kubernetes/apps/volsync-system/namespace.yaml`:
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: volsync-system
```
`kubernetes/apps/volsync-system/kustomization.yaml`:
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./namespace.yaml
  - ./volsync/ks.yaml
```
`kubernetes/apps/volsync-system/volsync/ks.yaml`:
```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: volsync
spec:
  interval: 1h
  path: ./kubernetes/apps/volsync-system/volsync/app
  prune: true
  sourceRef: {kind: GitRepository, name: flux-system, namespace: flux-system}
  targetNamespace: volsync-system
  wait: true
```
`kubernetes/apps/volsync-system/volsync/app/kustomization.yaml`:
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./ocirepository.yaml
  - ./helmrelease.yaml
```
`kubernetes/apps/volsync-system/volsync/app/ocirepository.yaml`:
```yaml
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: volsync
spec:
  interval: 1h
  layerSelector: {mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip, operation: copy}
  ref: {tag: <VOLSYNC_VER>}
  url: <VOLSYNC_OCI>
```
`kubernetes/apps/volsync-system/volsync/app/helmrelease.yaml`:
```yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: volsync
spec:
  interval: 1h
  chartRef: {kind: OCIRepository, name: volsync}
  values:
    manageCRDs: true
    metrics:
      disableAuth: true
```

- [ ] **Step 3: Write the reusable component**

`kubernetes/components/volsync/kustomization.yaml`:
```yaml
---
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
resources:
  - ./externalsecret.yaml
  - ./replicationsource.yaml
```
`kubernetes/components/volsync/externalsecret.yaml`:
```yaml
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: "${APP}-volsync-secret"
spec:
  secretStoreRef: {kind: ClusterSecretStore, name: onepassword}
  target:
    name: "${APP}-volsync-secret"
    template:
      data:
        RESTIC_REPOSITORY: "s3:https://932a695a6a0d664160ac575fa0e4fda8.r2.cloudflarestorage.com/kichi/volsync/${APP}"
        RESTIC_PASSWORD: "{{ .RESTIC_PASSWORD }}"
        AWS_ACCESS_KEY_ID: "{{ .AWS_ACCESS_KEY_ID }}"
        AWS_SECRET_ACCESS_KEY: "{{ .AWS_SECRET_ACCESS_KEY }}"
  dataFrom:
    - extract: {key: volsync}
    - extract: {key: <R2_ITEM>}
```
`kubernetes/components/volsync/replicationsource.yaml`:
```yaml
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: "${APP}"
spec:
  sourcePVC: "${APP}"
  trigger:
    schedule: "0 * * * *"
  restic:
    copyMethod: Direct
    repository: "${APP}-volsync-secret"
    pruneIntervalDays: 7
    retain:
      daily: 7
      weekly: 4
    moverSecurityContext:
      runAsUser: ${VOLSYNC_UID:=1000}
      runAsGroup: ${VOLSYNC_GID:=1000}
      fsGroup: ${VOLSYNC_GID:=1000}
```
(`Direct` mounts the live PVC read-only into the mover — no snapshot CRDs needed on hostpath. Apps whose PVC is not named after the app must set `sourcePVC` themselves; the phase-2 ports will follow the `${APP}` naming.)

- [ ] **Step 4: Commit, push, verify operator**

```bash
git add kubernetes/apps/volsync-system kubernetes/components/volsync && git commit -m "feat(volsync): operator + restic-to-R2 component" && git push && just kube reconcile
kubectl -n volsync-system get pods && kubectl get crd replicationsources.volsync.backube
```
Expected: `volsync-*` pod Running; CRD present.

- [ ] **Step 5: End-to-end backup test with a scratch app (kept until Task 1.8 restores it)**

```bash
kubectl create namespace scratch
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: probe, namespace: scratch}
spec: {accessModes: [ReadWriteOnce], resources: {requests: {storage: 1Gi}}}
---
apiVersion: v1
kind: Pod
metadata: {name: probe, namespace: scratch}
spec:
  securityContext: {runAsUser: 1000, runAsGroup: 1000, fsGroup: 1000}
  containers: [{name: w, image: busybox:1.36, command: [sh, -c, "date > /d/stamp && sleep 36000"], volumeMounts: [{name: d, mountPath: /d}]}]
  volumes: [{name: d, persistentVolumeClaim: {claimName: probe}}]
EOF
mkdir -p /private/tmp/claude-501/-Users-calvin-repo-kichi-org-home-ops/ade757b7-b18a-4827-b665-0f8b26076faf/scratchpad/probe
cd /private/tmp/claude-501/-Users-calvin-repo-kichi-org-home-ops/ade757b7-b18a-4827-b665-0f8b26076faf/scratchpad/probe
cp /Users/calvin/repo/kichi-org/home-ops-v2/kubernetes/components/volsync/*.yaml .
cat > kustomization.yaml <<'EOF'
resources: [externalsecret.yaml, replicationsource.yaml]
EOF
kubectl kustomize . | APP=probe VOLSYNC_UID=1000 VOLSYNC_GID=1000 envsubst | kubectl -n scratch apply -f -
kubectl -n scratch patch replicationsource probe --type merge -p '{"spec":{"trigger":{"manual":"now"}}}'
sleep 90; kubectl -n scratch get replicationsource probe -o jsonpath='{.status.lastSyncTime} {.status.latestMoverStatus.result}{"\n"}'
```
Expected: a timestamp and `Successful`. R2 now has `volsync/probe/` (check as in Task 1.4 Step 7).

### Task 1.6: kube-prometheus-stack, Alertmanager → Pushover, Flux alerts component

**Files:**
- Create: `kubernetes/apps/observability/{namespace.yaml,kustomization.yaml}`, `kube-prometheus-stack/{ks.yaml,app/*}` (ported), `kube-prometheus-stack/app/prometheusrule-backups.yaml` (new)
- Create: `kubernetes/components/alerts/**` (ported)
- Modify: `kubernetes/apps/external-secrets/kustomization.yaml` (components → alerts), `kubernetes/apps/database/kustomization.yaml`, `kubernetes/apps/volsync-system/kustomization.yaml` (add alerts component)

**Produces:** Prometheus (`prometheus.kichi.live` on envoy-internal, 7d/15GB on hostpath), Alertmanager with the Pushover receiver, VolSync + CNPG backup alerts wired.

- [ ] **Step 1: Port**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2
mkdir -p kubernetes/apps/observability && cp ../home-ops/kubernetes/apps/observability/namespace.yaml kubernetes/apps/observability/
cp -R ../home-ops/kubernetes/apps/observability/kube-prometheus-stack kubernetes/apps/observability/
cp -R ../home-ops/kubernetes/components/alerts kubernetes/components/alerts
grep -rn 'longhorn\|retentionSize\|retention:' kubernetes/apps/observability/kube-prometheus-stack/app/helmrelease.yaml
```
Expected: two `storageClassName: longhorn` (Prometheus, Alertmanager) and `retention: 7d` / `retentionSize: 16GB`.

- [ ] **Step 2: Adjust values**

In `kube-prometheus-stack/app/helmrelease.yaml`: replace both `storageClassName: longhorn` → `storageClassName: openebs-hostpath`; `retentionSize: 16GB` → `retentionSize: 15GB`; Prometheus storage request `20Gi` → `20Gi` (keep). Remove any `dependsOn: [{name: longhorn…}]` from `kube-prometheus-stack/ks.yaml`. Confirm the `valuesFrom` ConfigMap generator (`flux-metrics-configmap`) is still produced by `app/kustomization.yaml`.

`kubernetes/apps/observability/kustomization.yaml`:
```yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
components:
  - ../../components/alerts
resources:
  - ./namespace.yaml
  - ./kube-prometheus-stack/ks.yaml
```
Add `../../components/alerts` to the `components:` list of `database`, `volsync-system`, `openebs-system` and switch `external-secrets` back from `sops` to `alerts` (the `alerts` Provider/Alert are namespace-scoped Flux objects — one per namespace is the existing pattern).

- [ ] **Step 3: VolSync alert rule**

`kubernetes/apps/observability/kube-prometheus-stack/app/prometheusrule-backups.yaml`:
```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: volsync-backups
spec:
  groups:
    - name: volsync.backups
      rules:
        - alert: VolSyncOutOfSync
          expr: volsync_volume_out_of_sync == 1
          for: 2h
          labels: {severity: critical}
          annotations:
            summary: "VolSync {{ $labels.namespace }}/{{ $labels.obj_name }} has not synced within its schedule for 2h"
        - alert: VolSyncMissedIntervals
          expr: increase(volsync_missed_intervals_total[24h]) > 3
          labels: {severity: warning}
          annotations:
            summary: "VolSync {{ $labels.namespace }}/{{ $labels.obj_name }} missed >3 intervals in 24h"
```
Add it to `app/kustomization.yaml` resources. VolSync's metrics need a `ServiceMonitor`; add to `kubernetes/apps/volsync-system/volsync/app/helmrelease.yaml` values:
```yaml
    metrics:
      disableAuth: true
      serviceMonitor:
        enabled: true
```
(if the chart lacks that key, create a `ServiceMonitor` selecting the `volsync-metrics` service on port `https`/`metrics` — check `kubectl -n volsync-system get svc`.)

- [ ] **Step 4: Commit, push, verify**

```bash
git add kubernetes && git commit -m "feat(observability): kube-prometheus-stack + pushover + backup alerts" && git push && just kube reconcile
kubectl -n observability get pods,pvc && kubectl -n observability get alertmanagerconfig alertmanager
kubectl -n observability get secret alertmanager-secret -o jsonpath='{.data}' | jq 'keys'
```
Expected: `prometheus-kube-prometheus-stack-prometheus-0`, `alertmanager-…-0` Running with hostpath PVCs; `["ALERTMANAGER_PUSHOVER_TOKEN","PUSHOVER_USER_KEY"]`.

- [ ] **Step 5: Prove the Pushover path and the rules**

```bash
kubectl -n observability port-forward svc/alertmanager-operated 9093:9093 >/dev/null 2>&1 &
curl -s -XPOST localhost:9093/api/v2/alerts -H 'Content-Type: application/json' -d '[{"labels":{"alertname":"NewClusterTest","severity":"critical","cluster":"talos-11"},"annotations":{"summary":"Pushover path test from the new cluster"}}]'
kill %1
kubectl -n observability port-forward svc/prometheus-operated 9090:9090 >/dev/null 2>&1 &
curl -s 'localhost:9090/api/v1/rules' | jq -r '.data.groups[] | select(.name|test("backups")) | .rules[].name'
curl -s 'localhost:9090/api/v1/query?query=volsync_volume_out_of_sync' | jq '.data.result | length'
kill %1
```
Expected: Pushover notification arrives on Calvin's phone within ~1 min; rules `CNPGBackupStale`, `CNPGWALArchiveFailing`, `VolSyncOutOfSync`, `VolSyncMissedIntervals` listed; VolSync query returns ≥ 1 series (the `scratch/probe` source).

### Task 1.7: Grafana (operator + instance on hostpath, VolSync-backed)

**Files:**
- Create: `kubernetes/apps/observability/grafana/{ks.yaml,app/*,instance/*}` (ported)
- Modify: `kubernetes/apps/observability/kustomization.yaml` (+ `./grafana/ks.yaml`)

**Produces:** `grafana.kichi.live` on envoy-internal (reachable via `--resolve` / temporary Technitium record to `172.16.0.31`), datasources `prometheus`, `alertmanager`, `TeslaMate` (the latter unhealthy until phase 2 creates the DB), PVC `grafana` backed up hourly.

- [ ] **Step 1: Port and adjust**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2
cp -R ../home-ops/kubernetes/apps/observability/grafana kubernetes/apps/observability/grafana
grep -rn 'longhorn\|dependsOn' -A2 kubernetes/apps/observability/grafana/
```
Expected: `storageClassName: longhorn` in `instance/` and `dependsOn: [grafana, longhorn]` in `ks.yaml`. Change the storageClass to `openebs-hostpath`, ensure the PVC name is `grafana`, remove the `longhorn` dependency, and in `grafana-instance`'s Kustomization add:
```yaml
  components:
    - ../../../../components/volsync
  postBuild:
    substitute:
      APP: grafana
      VOLSYNC_UID: "472"
      VOLSYNC_GID: "472"
```
(472 = Grafana's uid; confirm against the `Grafana` CR's `securityContext`.) Add `./grafana/ks.yaml` to `observability/kustomization.yaml`.

- [ ] **Step 2: Commit, push, verify**

```bash
git add kubernetes/apps/observability && git commit -m "feat(grafana): operator + instance on hostpath with volsync" && git push && just kube reconcile
kubectl -n observability get grafana,pvc,replicationsource
curl -sk --resolve grafana.kichi.live:443:172.16.0.31 https://grafana.kichi.live/api/health
```
Expected: `Grafana grafana` stage `complete`; PVC `grafana` Bound; `ReplicationSource grafana` present; health JSON `"database":"ok"`. Dashboards from the operator's `GrafanaDashboard` CRs render (kube-prometheus-stack export).

### Task 1.8: Restore drills (both backup layers)

**Files:** none committed (scratch namespace only).

- [ ] **Step 1: VolSync restore of `scratch/probe` into a fresh PVC**

```bash
kubectl -n scratch delete pod probe
kubectl -n scratch apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: probe-restore}
spec: {accessModes: [ReadWriteOnce], resources: {requests: {storage: 1Gi}}}
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata: {name: probe-restore}
spec:
  trigger: {manual: restore-once}
  restic:
    repository: probe-volsync-secret
    destinationPVC: probe-restore
    copyMethod: Direct
    moverSecurityContext: {runAsUser: 1000, runAsGroup: 1000, fsGroup: 1000}
EOF
sleep 90; kubectl -n scratch get replicationdestination probe-restore -o jsonpath='{.status.latestMoverStatus.result}{"\n"}'
kubectl -n scratch run verify --rm -it --restart=Never --image=busybox:1.36 --overrides='{"spec":{"containers":[{"name":"v","image":"busybox:1.36","command":["cat","/d/stamp"],"volumeMounts":[{"name":"d","mountPath":"/d"}]}],"volumes":[{"name":"d","persistentVolumeClaim":{"claimName":"probe-restore"}}]}}'
```
Expected: `Successful`; the `stamp` content printed = the date written in Task 1.5.

- [ ] **Step 2: CNPG recovery from R2 into a scratch cluster**

```bash
kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -c "create database drill; \c drill \\ create table t as select now() as at;" 
sleep 120   # let WAL ship
kubectl -n scratch apply -f - <<'EOF'
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata: {name: cloudnative-pg-r2-secret}
spec:
  secretStoreRef: {kind: ClusterSecretStore, name: onepassword}
  target: {name: cloudnative-pg-r2-secret}
  dataFrom: [{extract: {key: <R2_ITEM>}}]
---
apiVersion: barmancloud.cnpg.io/v1
kind: ObjectStore
metadata: {name: r2}
spec:
  configuration:
    destinationPath: s3://kichi/cnpg/
    endpointURL: https://932a695a6a0d664160ac575fa0e4fda8.r2.cloudflarestorage.com
    s3Credentials:
      accessKeyId: {name: cloudnative-pg-r2-secret, key: AWS_ACCESS_KEY_ID}
      secretAccessKey: {name: cloudnative-pg-r2-secret, key: AWS_SECRET_ACCESS_KEY}
---
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata: {name: drill}
spec:
  instances: 1
  storage: {size: 5Gi, storageClass: openebs-hostpath}
  bootstrap:
    recovery:
      source: postgres
  externalClusters:
    - name: postgres
      plugin:
        name: barman-cloud.cloudnative-pg.io
        parameters:
          barmanObjectName: r2
          serverName: postgres
EOF
sleep 180; kubectl -n scratch get cluster drill
kubectl -n scratch exec drill-1 -c postgres -- psql -U postgres -d drill -tAc "select at from t"
```
Expected: `drill` `Cluster in healthy state`; the timestamp row is returned (i.e. WAL after the base backup was replayed).

- [ ] **Step 3: Clean up scratch and record the drill**

```bash
kubectl delete namespace scratch
kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -c "drop database drill"
```
Append to the Execution log below: date, VolSync restore OK, CNPG PITR OK.

### Task 1.9: Archive the template and close out phase 1

**Files:** `template/`, `makejinja.toml`, `cluster*.toml`, `pyproject.toml`, `uv.lock` → moved to `.private/<epoch>/` (gitignored) by `just template tidy`; `docs/superpowers/plans/…` execution log.

- [ ] **Step 1: Tidy**

```bash
cd /Users/calvin/repo/kichi-org/home-ops-v2 && just template tidy && git status --short | head
```
Expected: template files deleted from the tree, `justfile` trimmed to `bootstrap`/`kube`/`talos` modules; `cluster.toml` preserved under `.private/` (keep it — the MAC and IPs are there).

- [ ] **Step 2: Commit and tag**

```bash
git add -A && git commit -m "chore(template): tidy after bootstrap" && git push
git tag -a phase-1-foundation -m "Single-node rebuild: foundation complete (Talos, Flux, ESO, OpenEBS, CNPG+R2, VolSync+R2, Prometheus/Pushover, Grafana; restores drilled)" && git push origin phase-1-foundation
```

- [ ] **Step 3: Steady-state check against the spec's success criteria**

```bash
kubectl top node && kubectl get pods -A | grep -vE 'Running|Completed' ; kubectl get kustomization -A | grep -v True
```
Expected: node RSS well under 12 GiB; no non-Running pods; every Kustomization `Ready=True`.

- [ ] **Step 4: Update memory + hand off**

Record in the memory file `migration-design-doc` (or a new `migration-progress` note): phase 1 done on date, tag, node/VIP/gateway IPs live, the `home-ops-v2` worktree path, R2 item name, and that `just configure` must not be re-run. Next plan: phase 2 (ARC cutover) per spec section 8.

---

## Execution log

- Template commit: _(fill in Task 0.1)_
- SCHEMATIC_ID / TALOS_VER: _(Task 0.2)_
- VM 811 MAC: _(Task 0.3)_
- R2 1Password item: _(Task 1.4)_
- Restore drills: _(Task 1.8)_
