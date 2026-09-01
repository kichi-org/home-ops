# Kopiur evaluation — replacement for VolSync+restic

Research date: 2026-09-01. All findings from primary sources: the
[home-operations/kopiur](https://github.com/home-operations/kopiur) repo (docs/, Rust CRD source,
Helm chart, examples), its Releases/Issues via the GitHub API, and live manifests in
`onedr0p/home-ops` and `bjw-s-labs/home-ops`. No blog posts.

## Verdict

**GO on the decision-critical fact: the OpenEBS-hostpath / no-CSI-snapshot path is a
first-class, documented mode.** `SnapshotPolicy.spec.copyMethod: Direct` mounts the live
source PVC read-only into the mover with no VolumeSnapshot, no clone, no
VolumeSnapshotClass — explicitly documented as "works on ANY storage, including
local-path / hostPath with no snapshot support"
([docs/copy-methods.md](https://github.com/home-operations/kopiur/blob/main/docs/copy-methods.md),
[deploy/examples/23-copy-method-direct.yaml](https://github.com/home-operations/kopiur/blob/main/deploy/examples/23-copy-method-direct.yaml)).

Two must-knows for our cluster:

1. **`copyMethod` defaults to `Snapshot` since 0.5.0** (it used to be `Direct`). On a cluster
   with no CSI snapshot stack, a policy that omits `copyMethod` fails with
   `SnapshotStackMissing` — it never silently falls back to a live read. **Pin
   `copyMethod: Direct` explicitly on every SnapshotPolicy.**
2. Everything else in our VolSync workflow has a direct, mostly better equivalent:
   S3/R2 backend with documented R2 stanza, deploy-or-restore volume populator with
   `onMissingSnapshot: Continue` (same bootstrap pattern as VolSync
   ReplicationDestination + dataSourceRef), rich Prometheus metrics + a shipped
   PrometheusRule/dashboard (far beyond VolSync), auto-managed kopia maintenance, and a
   `kubectl kopiur migrate volsync` translator that maps restic ReplicationSources to
   kopiur manifests (config only — restic data cannot be carried over; repo starts empty).

Overall risk: the project is 3 months old (created 2026-06-02) with a very fast release
cadence and several breaking releases already; but it is under the home-operations org,
authored by perfectra1n (the VolSync-kopia fork maintainer) + onedr0p, both of whom run it
in their own production home-ops repos with the exact Flux component pattern we use.
Chart floor is `kubeVersion >=1.32.0-0`.

---

## 1. Snapshot-less direct PVC backup (decision-critical)

**Supported.** `CopyMethod` is a Rust enum `Snapshot | Clone | Direct`
(`crates/api/src/snapshot_policy.rs` L466-474). `volumeSnapshotClassName` is
`Option<String>` and only used when copyMethod snapshots/clones the source — the field is
irrelevant under `Direct`. onedr0p sets `volumeSnapshotClassName: csi-ceph-blockpool`
because he runs Ceph with the default `Snapshot` mode; that is his storage choice, not a
kopiur requirement.

Direct-mode behavior ([docs/copy-methods.md § Direct](https://github.com/home-operations/kopiur/blob/main/docs/copy-methods.md)):

- Mounts the **live** source PVC read-only; crash-consistent (same semantics as VolSync
  `copyMethod: Direct`).
- For RWO volumes the mover is auto co-located on the node holding the PVC (multi-attach
  avoidance) — a no-op on our single node.
- If the snapshot stack is missing and `copyMethod` was left defaulted, the backup fails
  with a clear condition (`SnapshotStackMissing` / `NoVolumeSnapshotClass` /
  `SourceNotCSIProvisioned`) telling you to set `Direct`.
- App-consistency via `spec.hooks` (beforeSnapshot/afterSnapshot exec in the workload pod)
  if we ever want quiesced DB backups; not required.
- Sharp edge: `Direct` + `sources[].readOnly: false` rewrites live data via fsGroup chown —
  rejected at admission unless `acknowledgeLiveMutation: true`. Don't set `readOnly: false`.
- GitOps hazard called out in docs/copy-methods.md: a server-defaulted `copyMethod` has no
  field owner under SSA, so an omitted value can silently flip to `Snapshot` on re-apply —
  another reason to always pin it.

## 2. S3 / Cloudflare R2 backend

[docs/backends/s3.md](https://github.com/home-operations/kopiur/blob/main/docs/backends/s3.md)
names **Cloudflare R2 explicitly** as a supported S3-compatible store, with a per-provider
table (`endpoint: <accountid>.r2.cloudflarestorage.com`, `region: auto`, "use an R2 API
token's S3 credentials") and a filled-in R2 stanza. No open or closed issues mention R2
problems. Secret keys (loaded via `envFrom`): `AWS_ACCESS_KEY_ID`,
`AWS_SECRET_ACCESS_KEY`, `KOPIA_PASSWORD` (mandatory everywhere; unrecoverable if lost).
`spec.parameters.blobRetention` can turn on S3 object lock for ransomware protection.

## 3. Prometheus metrics / alerting

Much stronger than VolSync:

- ~50 metrics under `kopiur_` documented in
  [docs/dev/observability.md](https://github.com/home-operations/kopiur/blob/main/docs/dev/observability.md).
  Key backup-health ones: `kopiur_snapshotpolicy_last_backup_success`,
  `kopiur_policy_last_backup_success_timestamp_seconds`,
  `kopiur_snapshot_consecutive_failures`, `kopiur_snapshots_completed_total{result}`.
- Chart ships `monitoring.serviceMonitor`, `monitoring.prometheusRule` (alerts:
  `KopiurBackupConsecutiveFailures`, `KopiurBackupStale` — covers the never-succeeded
  gap, `KopiurLastBackupFailed` recovery-aware, `KopiurSnapshotFailed`,
  `KopiurRestoreFailed`, repository-breaker alerts) and Grafana dashboards
  (ConfigMap-sidecar or grafana-operator CR). Alert rules have their own unit tests.
- Both onedr0p and bjw-s simply enable these chart values — no custom kube-state-metrics
  config needed. Slots straight into our kube-prometheus-stack → Alertmanager → Pushover
  pipeline.

## 4. Restore workflow / bootstrap

Same deploy-or-restore pattern as VolSync, cleaner
([docs/restores.md](https://github.com/home-operations/kopiur/blob/main/docs/restores.md)):

- A passive `Restore` with `source.fromPolicy: {name, offset: 0}` + `target.populator: {}`
  is claimed by the PVC's `dataSourceRef` (AnyVolumeDataSource, k8s >= 1.24).
- **Fresh cluster / empty repo:** `policy.onMissingSnapshot: Continue` is the _default_
  for `fromPolicy` sources — kopiur provisions an empty prime PVC and rebinds it so the
  app starts; the decision is pinned to `status.resolved` so a later snapshot never
  silently overwrites a live volume.
- The populator Restore is **reusable**: delete the PVC and reapply, it repopulates; a
  bound claim is a no-op (`TargetAlreadyBound`).
- **Point-in-time:** `source.fromPolicy.asOf: <RFC3339>` or `offset: N`; also
  `source.snapshotRef` / raw `source.identity` + `snapshotID`.
- Flux-friendliness is explicit: kstatus conditions on every CRD, healthChecks/CEL
  examples, status-only writes, materialized defaults to avoid diff-thrash
  ([docs/gitops.md](https://github.com/home-operations/kopiur/blob/main/docs/gitops.md)).

## 5. Retention & maintenance

- **Retention** (`SnapshotPolicy.spec.retention`, GFS): `keepLatest`, `keepHourly`,
  `keepDaily`, `keepWeekly`, `keepMonthly`, `keepAnnual`. A policy **without**
  `spec.retention` never prunes (deliberate safe default). Snapshot CRs own their kopia
  snapshots; `deletionPolicy` defaults to `Delete` (Retain/Orphan available), with a
  mass-deletion circuit breaker (ADR-0006).
- **Maintenance is automatic — no CR required.** Every Repository gets an
  operator-projected `Maintenance` (quick every 6h, full GC daily 03:00, jittered);
  tunable inline or disable with `enabled: false`.

## 6. Maturity & cadence

- Created **2026-06-02**; current release **0.10.5** (2026-08-28); AGPL-3.0.
- **62 releases in 3 months**; release-please conventional-commit automation.
- **Breaking changes** (CHANGELOG.md): 0.5.0 (**copyMethod default flipped
  Direct→Snapshot**), 0.6.0 (chart values regrouped; CRDs moved to Helm `crds/` — a
  one-time destructive migration under GitOps, documented in docs/upgrade.md), 0.9.0
  (security-context/identity rework), 0.10.0 (new RBAC, apply CRDs before operator).
  Expect more pre-1.0 CRD churn; read release notes on every Renovate bump.
- **Issues:** 65 total, only 4 real open (2 enhancements, 1 flaky e2e). Closed bug themes
  were operational leaks — all fixed with post-mortem-grade changelog notes.
- **Contributors:** perfectra1n (maintainer of the perfectra1n/volsync kopia fork),
  onedr0p, plus bot/AI-assisted automation. Effectively a 2-human project.
- **Test signals:** CI runs cargo audit, Codecov, and a sharded kind-based e2e suite
  (~40 e2e files incl. copy_methods.rs, restore.rs, retention.rs); Helm chart has
  unittest files incl. alert-rule tests.

## 7. Runtime footprint

- **Controller Deployment** (1 replica; requests 50m/128Mi default) + **admission webhook**
  (25m/64Mi) + **per-operation mover Jobs** in the workload namespace (short-lived,
  ttl 1h). Kopia cache per mover: emptyDir by default; sized/persistent cache opt-in.
- **No central repository server required** (kopia web UI is opt-in via `spec.server`).
- 8 CRDs; `kubeVersion >= 1.32` floor. Entirely reasonable for a single-node 24GB cluster.

## 8. Migration surface from VolSync

- **`kubectl kopiur migrate volsync`**: translates ReplicationSource/Destination →
  SnapshotPolicy/SnapshotSchedule/Restore, offline from Git files or live. For
  upstream-VolSync restic (us): **config translation ONLY — repo formats are
  incompatible, the kopia repository starts empty. Keep the restic R2 repos until kopiur
  retention coverage suffices.**
- **Mover security context:** default mover runs as UID 65532; app data owned by other
  UIDs needs `spec.mover.securityContext.runAsUser/runAsGroup` per policy (template
  `KOPIUR_PUID/GID` exactly like VolSync's `runAsUser`). Root movers require namespace
  annotation `kopiur.home-operations.com/privileged-movers: "true"`.
- **Credentials must exist in each workload namespace** (movers run where the PVC is).
  Options: per-namespace ExternalSecret (what both reference repos do — matches our
  1Password+ESO setup), ClusterRepository `credentialProjection`, or a namespaced
  Repository per app.
- **ClusterRepository tenancy gate:** `spec.allowedNamespaces` is mandatory-shaped;
  both reference repos use `all: true`.
- **Identity matters:** kopia snapshots are keyed `username@hostname:path`; pin
  `identityDefaults: {hostnameExpr: namespace, usernameExpr: policyName}` on the
  ClusterRepository before first backups — changing identity later strands history.
- Deletion semantics differ from VolSync: `Snapshot` CRs are durable records that own
  the kopia snapshots; deleting one deletes the backup by default.

## Open questions

1. **Volume populator + OpenEBS hostpath (WaitForFirstConsumer):** the prime-PVC rebind
   flow is unproven against the non-CSI `openebs.io/local` provisioner (both reference
   users run CSI storage). **Needs a smoke test on our cluster before committing**;
   restore into a fresh PVC via `target.pvc` is the fallback. The one remaining go/no-go
   verification.
2. Mover behavior reading a WaitForFirstConsumer-bound hostpath PVC — expected fine on a
   single node (co-location automatic), but confirm the first Direct backup schedules.
3. `status.full.lastContentReclaimedBytes` currently always 0 (documented limitation).
4. AGPL-3.0 license — fine for homelab use.
5. Pre-1.0 CRD churn: budget for reading release notes on every Renovate bump; the
   0.5→0.6 CRD relocation showed an upgrade can cascade-delete CRs if mishandled.
