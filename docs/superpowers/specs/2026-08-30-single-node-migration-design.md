# Single-node Talos migration — design

Status: approved 2026-08-30 (open decisions resolved, see section 12)
Inputs: discovery + decisions from the Aug 2026 sessions (UniFi, Proxmox, Cloudflare, GitHub, cluster load, gap analysis vs cluster-template `b5f9619`).

## 1. Goal

Replace the 3-node Talos cluster (talos-1..3 on kl-vhost-1) with a **fresh single-node Talos cluster** generated from the current onedr0p/cluster-template, with Longhorn replaced by OpenEBS LocalPV-hostpath and **backups working from day one**. Family services keep running throughout: every stateful/singleton service is cut over individually with a rollback path.

### Non-goals

- Shrinking the existing cluster in place.
- HA of any kind (one host = one failure domain; accepted).
- Changing the virtual NAS, the Win11 jump host, adding 10 GbE or a third host.
- Immich (never deployed, no data — directory removed in the rebuild).
- eBGP, Server firewall zone, IoT VLAN, identity provider — all **after** the new cluster is stable (section 11).

## 2. Current state (summary)

| Item              | Today                                                                                                                                                                                                              |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Cluster           | 3× control-plane Talos v1.13.9 / K8s v1.35.0, VMs 801–803 on kl-vhost-1 (4 vCPU / 12 GiB / 32 G + 100 G each, `adata-nvme`, disks `backup=0`)                                                                      |
| Network           | VLAN 100 `172.16.0.0/24`, VIP `.10`, nodes `.101–.103`, LB-IPAM = whole /24; pinned `.11` k8s-gateway, `.12` envoy-internal, `.13` envoy-external, `.122` cnpg, `.123` qbittorrent, `.128` plex, `.200` technitium |
| Storage           | Longhorn (2.6 GiB RAM, 0 backups in 211 d); bulk media on NFS `kl-san-1:/volume1/data`                                                                                                                             |
| Load              | ~1 CPU core, 15.5 GiB working set, of which ≈ 8.5 GiB is triplicated control plane + Longhorn                                                                                                                      |
| Data that matters | ≈ 8 GiB: TeslaMate PG 1.4, Plex 1.7, Technitium 1.3, *arr ~1.5, Grafana 0.6                                                                                                                                        |
| Backups           | none (Proxmox or Kubernetes)                                                                                                                                                                                       |
| Public surface    | Cloudflare Tunnel `kubernetes` → plex, flux-webhook (echo to be removed)                                                                                                                                           |
| Secrets           | 1Password Connect + ESO for apps; SOPS/age for bootstrap secrets                                                                                                                                                   |
| Ops               | Renovate PRs merged by hand → Flux/tuppr apply; Pushover alerts                                                                                                                                                    |

## 3. Target architecture

### 3.1 Host and VM

|             | Value                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| Build host  | **kl-vhost-2** (58 GB idle) — old cluster untouched during transition                                                                |
| Final host  | **kl-vhost-1** (`adata-nvme` thin pool), moved via vzdump/qmrestore after the last cutover (section 9)                               |
| VMID / name | `811` / `talos-11` (8xx = Talos convention)                                                                                          |
| CPU / RAM   | 8 vCPU `cpu: host`, **24 GiB**, no ballooning (16 GiB floor; measured need ≈ 9.5 GiB + page cache)                                   |
| Disks       | `scsi0` 50 G install disk, `scsi1` **200 G** data disk — both **`backup=1`** (the old `backup=0` flag is why vzdump would skip them) |
| NIC         | `vmbr0`, `tag=100`, fixed MAC recorded in `cluster.toml`                                                                             |
| Misc        | qemu-guest-agent on, start at boot after NAS (order 2, NAS is 1 with 240 s delay)                                                    |

### 3.2 Talos

- Image Factory schematic with **only `qemu-guest-agent`** (iscsi-tools/util-linux were Longhorn-only).
- Secrets **regenerated** by topf (no talhelper import).
- Flat interface config (no bond); `vlan_tag` unset — Proxmox bridge tags the traffic.
- Patches carried from home-ops:
    - `UserVolumeConfig` `data` on the 200 G disk → `/var/mnt/data` (OpenEBS basePath `/var/mnt/data/openebs`).
    - NFS `nfsmount.conf` (nfsvers=4.1, nconnect=16).
    - `kubernetesTalosAPIAccess` for tuppr (+ ARC if it still needs it).
- `allowSchedulingOnControlPlanes` = on (template default for 1 node). Add a `[[nodes]]` entry later for the iGPU worker on kl-vhost-2 — nothing in the design assumes "exactly one node" beyond replica counts.

### 3.3 Addressing (transition vs final)

Same VLAN 100, disjoint addresses; canonical addresses are taken over one singleton at a time.

| Role           | Old cluster | New — transition | New — final                                           |
| -------------- | ----------- | ---------------- | ----------------------------------------------------- |
| Node           | `.101–.103` | `.111`           | `.111` (renumber to `.101` optional)                  |
| Kube API VIP   | `.10`       | `.20`            | `.20` (or `.10` optional)                             |
| LB-IPAM pool   | whole /24   | `.30–.99`        | `.30–.99` + pinned canonical IPs                      |
| k8s-gateway    | `.11`       | `.32`            | `.11`                                                 |
| envoy-internal | `.12`       | `.31`            | `.12`                                                 |
| envoy-external | `.13`       | `.33`            | `.13`                                                 |
| Technitium     | `.200`      | `.34`            | `.200`                                                |
| Plex           | `.128`      | `.35`            | `.128`                                                |
| qBittorrent    | `.123`      | `.36`            | `.123`                                                |
| CNPG rw        | `.122`      | pool             | `.122` (only if anything outside the cluster uses it) |

DHCP on VLAN 100 is shrunk to `.220–.254` (Talos install pool) before the VM is created; the stale DHCP DNS option is fixed to `.200` + `192.168.85.1`.

### 3.4 Platform components

| Layer         | Component                                                                                                                                                           | Source           |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------- |
| CNI / LB      | Cilium, L2 announcements, LB-IPAM (BGP fields left blank)                                                                                                           | template         |
| GitOps        | flux-operator + flux-instance, webhook receiver `flux-webhook.kichi.live`                                                                                           | template         |
| Ingress       | envoy-gateway (internal/external), cloudflared tunnel `kubernetes`, external-dns (cloudflare)                                                                       | template         |
| DNS           | k8s-gateway (template) + Technitium + `technitium-dns` external-dns (RFC2136)                                                                                       | carry            |
| Certs         | cert-manager, Let's Encrypt DNS-01 wildcard (`shortlived` profile ok)                                                                                               | template         |
| Secrets       | ESO + 1Password Connect (ClusterSecretStore, vault `Kubernetes`)                                                                                                    | carry            |
| Storage       | **OpenEBS LocalPV-hostpath** (single StorageClass `openebs-hostpath`, default)                                                                                      | new              |
| Databases     | CloudNativePG; backups via **Barman Cloud plugin** (in-tree `barmanObjectStore` is deprecated)                                                                      | carry + change   |
| PVC backup    | VolSync + restic, `copyMethod: Direct` (no snapshot controller needed)                                                                                              | new              |
| Upgrades      | tuppr (Talos + K8s), Renovate self-hosted app                                                                                                                       | template / carry |
| Observability | kube-prometheus-stack (1 replica, 7 d / 15 Gi retention), grafana-operator + Grafana, Alertmanager → Pushover; VictoriaLogs optional, Gatus later on envoy-internal | carry, trimmed   |
| Dropped       | Longhorn, descheduler, echo, Immich, `kichi.internal`, dead `ceph-block` ref, aqua/mise/Taskfile                                                                    | —                |

Prometheus is "nice-to-have" but it is the alert path for Pushover and for the backup alerts below, so it stays with reduced retention.

### 3.5 Applications

Must-have: Plex, TeslaMate (+ Grafana dashboards), Sonarr/Radarr/Prowlarr, qBittorrent, qui, Recyclarr, Dispatcharr, Technitium, ARC (home-ops + home-labs runner sets).
Every app's PVC moves to `openebs-hostpath`; media stays on NFS (`kl-san-1:/volume1/data`, export ACL already covers `.111`).

## 4. Repository strategy

The template renders to the repo root (`kubernetes/`, `talos/`, `bootstrap/`, `cluster.toml`, `justfile`), which collides with the live tree that the old cluster's Flux reads.

**Decision proposed: orphan branch `v2` in `kichi-org/home-ops`.**

- New cluster's Flux watches branch `v2`; old cluster keeps `main` untouched.
- Renovate is repointed to `v2` only (old cluster is frozen for the few weeks it survives). CI (flux-local diff/test) runs on `v2` PRs.
- GitHub webhook → `flux-webhook.kichi.live` reaches whichever cluster owns the tunnel; the other polls (1 min). Fine.
- At retirement: tag `main` as `v1-final`, then `git push origin v2:main --force`, delete `v2`, flip the new cluster's Flux branch back to `main`. History of the old repo survives under the tag.

Alternative considered: a new repo `home-ops-v2` — cleaner Renovate but needs a second ARC scale set, webhook and App install, plus a rename at the end. Not worth it for a transition of weeks.

Layout inside `v2`: template output as-is, plus `kubernetes/apps/<ns>/<app>` ports. Singleton apps are committed **with their Flux Kustomization commented out / `suspend: true`** until their cutover PR.

## 5. Secrets and bootstrap

| Secret                                                                               | Source in rebuild                                                                                                                                                                  |
| ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Talos secrets                                                                        | regenerated by topf, `talos/talsecret` encrypted with the existing `age.key`                                                                                                       |
| `age.key`, Cloudflare DNS token, tunnel `cloudflare-tunnel.json`, flux webhook token | reused (already in SOPS files / local files)                                                                                                                                       |
| Flux git access                                                                      | none needed (repo public, `https://` URL)                                                                                                                                          |
| 1Password Connect seed (`onepassword-secret`: credentials.json + token)              | pulled with `op` from item `1password` in vault `Kubernetes`, `kubectl create secret` by hand right after bootstrap; same Connect server shared by both clusters during transition |
| Everything else (Pushover, R2 keys, TeslaMate, *arr API keys…)                       | ESO from 1Password — no SOPS for app secrets                                                                                                                                       |

Bootstrap order: `just` render → `talosctl apply` (node picks up a `.220–.254` DHCP lease in maintenance mode) → `talosctl bootstrap` → Cilium/CoreDNS/spegel-off/flux-operator via the template's bootstrap → create `onepassword-secret` → Flux reconciles `v2`.

## 6. Storage and backups

### 6.1 Layout

- 200 G data disk → `/var/mnt/data`; OpenEBS hostpath under `/var/mnt/data/openebs`.
- Prometheus TSDB and VictoriaLogs also on hostpath but **excluded** from backups.
- R2 bucket `kichi` (10 GB): prefixes `cnpg/<cluster>/` and `volsync/<app>/`. Old Longhorn never wrote anything there, so no conflict.

### 6.2 Backup layers

1. **CNPG → R2**: continuous WAL archiving + daily base backup, retention 7 d (TeslaMate, Grafana if it uses PG).
2. **VolSync/restic → R2**: Plex, Technitium, Sonarr, Radarr, Prowlarr, qBittorrent, Dispatcharr, Grafana — hourly, keep daily 7 / weekly 4. Each app ships with its `ReplicationSource` from the first commit ("backup from day one").
3. **vzdump** of VM 811 weekly, snapshot mode, keep 2, to `san-iso` (`/volume1/proxmox`) — disaster floor. Scheduled once the VM is on kl-vhost-1; on kl-vhost-2 (`kingston-nvme` is thick LVM, no snapshots) run it in stop or suspend mode if wanted before the move.

### 6.3 Alerting

PrometheusRules → Alertmanager → Pushover:

- VolSync: last successful sync > 24 h, or failed.
- CNPG: last base backup > 36 h, WAL archive failing.
- Restore test: after phase 1, restore one VolSync app and one CNPG cluster into a scratch namespace to prove the path before any cutover.

R2 usage reviewed after one month against the 10 GB quota (expected ≈ 8 GiB data + WAL churn — tight; Plex metadata is the candidate to trim if needed).

## 7. Networking, DNS and ingress during the transition

- **cert-manager** issues the same `*.kichi.live` wildcard on both clusters — DNS-01 TXT records are additive, no conflict.
- **Reaching new-cluster apps before DNS cutover:** add temporary records in the old Technitium (`<app>.kichi.live → .31`) or `dig @172.16.0.32` for verification. No `/etc/hosts` hacks needed.
- **DNS cutover is late** (section 8, step 7): when Technitium `.200` and k8s-gateway `.11` move, all `*.kichi.live` resolution switches to the new envoy-internal, so every app must already live there.
- **Cloudflare**: one PR switches cloudflared + `external-dns` (cloudflare) + Plex together. `external.kichi.live` CNAME stays on tunnel `kubernetes` (same tunnel, new connector). Mail/iCloud records untouched (external-dns owner TXT filter).
- **Port forwards** (UniFi, manual): 50469 → qBittorrent `.123`, new 32400 → Plex `.128` — re-pinned IPs mean **the forwards don't change**, only the cluster answering.
- **Plex direct**: `plex-direct.kichi.live` grey A record created by hand; UniFi DDNS entry repointed from apex to it; `PLEX_ADVERTISE_URL` += `https://plex-direct.kichi.live:32400`.

## 8. Migration sequence

Each step is one or more PRs on `v2`; each singleton cutover is **exactly one PR** that (a) suspends/removes the app on `main`, (b) enables it on `v2`, (c) re-pins the canonical LB IP.

| #   | Step                                                                                                                                                                                                                                  | Rollback                                                                                 |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| 0   | Prep: shrink VLAN100 DHCP to `.220–.254`, fix DHCP DNS option, create `plex-direct` record, cut `v2` branch from template, repoint Renovate                                                                                           | trivial                                                                                  |
| 1   | **Foundation**: create VM 811 on kl-vhost-2, bootstrap Talos/Flux, ESO, OpenEBS, CNPG, VolSync, cert-manager, envoy gateways on `.31/.33`, k8s-gateway `.32`, kube-prometheus-stack, Grafana. Verify a restore of both backup layers. | delete VM                                                                                |
| 2   | **ARC** (both scale sets; home-labs depends on it)                                                                                                                                                                                    | revert PR                                                                                |
| 3   | **TeslaMate + Grafana**: stop old TeslaMate → CNPG `bootstrap.initdb.import` from old cluster's PG (`.122`) → start new; import Grafana dashboards/DB                                                                                 | revert PR, old PG untouched                                                              |
| 4   | ***arr + qBittorrent + qui + Recyclarr**: stop old, tar config dirs to NFS staging (`/volume1/data/_migration`), untar into hostpath PVCs, start new; qBittorrent → `.123`                                                            | revert PR                                                                                |
| 5   | **Dispatcharr** (same rsync pattern)                                                                                                                                                                                                  | revert PR                                                                                |
| 6   | **Plex + Cloudflare**: stop old Plex, copy `Library/…` metadata (no rescan), enable cloudflared + external-dns on new, disable on old, Plex → `.128`, add 32400 forward                                                               | revert PR (tunnel reconnects to old within a minute)                                     |
| 7   | **DNS**: Technitium (export/import backup, → `.200`), `technitium-dns` external-dns, k8s-gateway → `.11`, envoy-internal → `.12`, envoy-external → `.13`                                                                              | revert PR; gateway NS record still points at `.11` so only the answering cluster changes |
| 8   | Soak 1–2 weeks with old cluster **powered off** (not deleted)                                                                                                                                                                         | power on, revert last PRs                                                                |
| 9   | **Retire**: delete VMs 801–803, tag `v1-final`, `v2 → main`, Renovate/Flux back to `main`                                                                                                                                             | tag                                                                                      |
| 10  | **Move** VM 811 to kl-vhost-1: shutdown → vzdump to `san-iso` → `qmrestore` to `adata-nvme` → boot (same MAC/IPs; offline because of Skylake→Zen `cpu: host`). This is also the rehearsal of the weekly vzdump/restore.               | boot the original on kl-vhost-2                                                          |
| 11  | Schedule weekly vzdump on kl-vhost-1; set startup order; cleanup unused `vm-101-disk-0` volumes                                                                                                                                       | —                                                                                        |

Data migration mechanics: a throwaway `migrate` pod per side (`alpine` + `tar`/`rsync`) mounting the source PVC (old, Longhorn) and the NFS staging dir, then the target PVC (new, hostpath). Apps are scaled to 0 on both sides during the copy; copies are re-runnable so a first pass can warm the staging dir while the old app still runs, then a short final delta with the app stopped.

## 9. Cutover invariants

- Only one cluster runs a given singleton at any time: cloudflared, both external-dns instances, Technitium, k8s-gateway on `.11`, TeslaMate (Tesla API session), ARC, qBittorrent (port forward).
- Old PVCs/DBs are never deleted before step 9.
- Every cutover PR is self-contained and revertible with `git revert` + Flux reconcile.
- Cilium L2 on both clusters must never announce the same IP — the pool split (`.30–.99` vs canonical) plus "disable old before enable new" in each PR guarantees it.

## 10. Success criteria

- All must-have apps reachable by their existing names; Plex streams via tunnel and via `plex-direct:32400`.
- TeslaMate history intact (row counts match), Grafana dashboards render.
- VolSync and CNPG backups green in Prometheus; one restore of each proven.
- Weekly vzdump present on `san-iso`, restore rehearsed by step 10.
- Node steady state ≤ 12 GiB RSS, no OOM; old VMs deleted; `main` = new repo.

## 11. After the migration (separate specs)

1. Server firewall zone on VLAN 100 (LAN→Server allow; Server→NAS NFS + established; Server→Internet).
2. IoT VLAN 20 + mDNS for Sonos.
3. eBGP Cilium ↔ UCG (`[cilium.bgp]`), L2 kept as fallback, pinned IPs never move.
4. Identity provider evaluation and Cloudflare Access re-target for the `mytunnel_iph` hostnames.
5. Optional renumber node/VIP to `.101`/`.10`; second node on kl-vhost-2 with iGPU passthrough.

## 12. Decisions taken 2026-08-30

1. **Repo**: orphan branch `v2` in `kichi-org/home-ops` (section 4), not a new repo.
2. **VictoriaLogs**: left out of the rebuild; add later only if missed.
3. **Grafana**: SQLite on hostpath, VolSync-backed (dashboards are provisioned by grafana-operator).
4. **Install disk**: 50 G (thin-provisioned).
5. **VM**: VMID `811`, hostname `talos-11`.
