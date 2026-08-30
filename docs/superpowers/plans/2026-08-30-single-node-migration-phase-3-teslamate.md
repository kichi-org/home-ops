# Single-node migration — Phase 3 (TeslaMate + Grafana) implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Move TeslaMate and its Postgres database (393 MB) from the old cluster to `talos-11` with no gap in vehicle history, and have the Grafana TeslaMate dashboards on the new cluster read from the migrated database.

**Architecture:** TeslaMate is a singleton (one Tesla API session per car), so the old deployment is stopped first (`main` PR, Flux prune), the database is copied old → new with `pg_dump | pg_restore` run from inside the new cluster's Postgres pod (the old primary is reachable on the `.122` LoadBalancer), and only then is TeslaMate enabled on `v2`. Its `postgres-init` init container is idempotent, so it tolerates the pre-created role/database. Grafana on `v2` already has the `TeslaMate` datasource pointing at `postgres-rw.database`; the 20 TeslaMate `GrafanaDashboard` CRs travel with the app.

**Tech Stack:** Flux, bjw-s app-template 5.1.0, `teslamate/teslamate:4.2.0`, `ghcr.io/home-operations/postgres-init:18`, CloudNativePG (Postgres 18.1) on both clusters, ESO + 1Password (items `teslamate`, `cloudnative-pg`), grafana-operator.

**Spec:** `docs/superpowers/specs/2026-08-30-single-node-migration-design.md` (§3.5, §6.2 layer 1, §8 step 3, §9). Predecessors: phase 0–1 and phase 2 plans in this directory.

## Global constraints

- Only one TeslaMate instance talks to Tesla at any time (spec §9): stop the old one before the new one starts.
- Old database is never modified or deleted; it stays on the old cluster until spec §8 step 9.
- Every change is a PR: `main` for the old cluster, `v2` for the new; Calvin merges. Commits: subject line only, no body, no trailers.
- New cluster tooling: `cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig`; commit with `mise exec -- git commit`. Old cluster: `cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig`.
- Secrets are read from the old cluster's Kubernetes Secrets (`teslamate-secret`, `cloudnative-pg-secret`) into shell variables and piped into commands — never printed.
- When removing an app from the old cluster, only the app is removed; its database and Secrets in other namespaces stay.

---

## File map

Branch `v2` (worktree `~/repo/kichi-org/home-ops-v2`):

| Path                                                                                                                         | Responsibility                                                                                 |
| ---------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `kubernetes/apps/observability/teslamate/ks.yaml`                                                                            | Flux Kustomization `teslamate` (depends on `cloudnative-pg-cluster`, `onepassword`, `grafana`) |
| `kubernetes/apps/observability/teslamate/app/{kustomization,ocirepository,helmrelease,externalsecret,grafanadashboard}.yaml` | copied from `main`; the dashboard file gets its corrupted URL fixed                            |
| `kubernetes/apps/observability/kustomization.yaml`                                                                           | + `./teslamate/ks.yaml`                                                                        |
| `docs/superpowers/plans/2026-08-30-single-node-migration-phase-3-teslamate.md`                                               | this plan                                                                                      |

Branch `main` (`~/repo/kichi-org/home-ops`): delete `kubernetes/apps/observability/teslamate/` and its line in `kubernetes/apps/observability/kustomization.yaml`. Old Grafana keeps its datasource to the old DB (static data) — untouched.

---

### Task 1: Port TeslaMate to `v2` on a feature branch and open the PR (not merged yet)

**Files:** see file map.

**Produces:** PR "feat(teslamate): port to the new cluster" against `v2`, left open. Record as `PR_V2`.

- [ ] **Step 1: Branch, copy, fix the dashboard URL, wire the Kustomization**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH
git checkout -b feat/teslamate v2
cp -R ../home-ops/kubernetes/apps/observability/teslamate kubernetes/apps/observability/teslamate
cp ../home-ops/docs/superpowers/plans/2026-08-30-single-node-migration-phase-3-teslamate.md docs/superpowers/plans/
sed -i '' 's|raw.github§usercontent.com|raw.githubusercontent.com|' kubernetes/apps/observability/teslamate/app/grafanadashboard.yaml
grep -c 'raw.githubusercontent.com/teslamate-org' kubernetes/apps/observability/teslamate/app/grafanadashboard.yaml
python3 - <<'EOF'
p='kubernetes/apps/observability/teslamate/ks.yaml'; s=open(p).read()
old='  name: teslamate\nspec:\n  interval: 1h\n'
new='''  name: teslamate
spec:
  dependsOn:
    - name: cloudnative-pg-cluster
      namespace: database
    - name: onepassword
      namespace: external-secrets
    - name: grafana
  interval: 1h
'''
assert old in s; open(p,'w').write(s.replace(old,new))
p='kubernetes/apps/observability/kustomization.yaml'; s=open(p).read()
assert './teslamate/ks.yaml' not in s
s=s.replace('  - ./kube-prometheus-stack/ks.yaml\n','  - ./kube-prometheus-stack/ks.yaml\n  - ./teslamate/ks.yaml\n'); open(p,'w').write(s)
EOF
kubectl kustomize kubernetes/apps/observability >/dev/null && kubectl kustomize kubernetes/apps/observability/teslamate/app | grep -E '^kind:' | sort | uniq -c
```

Expected: `20` dashboard URLs all well-formed; kinds: 1 ExternalSecret, 20 GrafanaDashboard, 1 HelmRelease, 1 OCIRepository.

- [ ] **Step 2: Pre-flight on the new cluster (read-only)**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl -n database get cluster postgres --no-headers
kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -tAc "select datname from pg_database where datname='teslamate'" | grep -c teslamate || echo "no teslamate db yet (expected)"
kubectl -n observability get secret grafana-datasource-password-secret -o jsonpath='{.data}' | jq -c 'keys'
kubectl -n database exec postgres-1 -c postgres -- bash -c 'exec 3<>/dev/tcp/172.16.0.122/5432 && echo "old primary reachable on .122:5432"'
```

Expected: cluster healthy; no `teslamate` DB yet; `["TESLAMATE_DATABASE_PASS"]`; old primary reachable from inside the new Postgres pod.

- [ ] **Step 3: Commit, push, open the PR**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH
git add kubernetes/apps/observability docs && mise exec -- git commit -q -m "feat(teslamate): port to the new cluster" && git push -u origin feat/teslamate
```

GitHub MCP `create_pull_request` (head `feat/teslamate`, base `v2`), body: "Phase 3 — enables TeslaMate on talos-11 against the migrated `teslamate` database. Merge only after the DB import (plan Task 3) is verified." **Do not merge yet.**

### Task 2: Stop TeslaMate on the old cluster (`main` PR, merged first)

**Files:** `main`: delete `kubernetes/apps/observability/teslamate/`, edit `kubernetes/apps/observability/kustomization.yaml`.

- [ ] **Step 1: Record the source-of-truth counts (old DB, TeslaMate still running)**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
OLDP=$(kubectl -n database get pods -l cnpg.io/cluster=postgres,cnpg.io/instanceRole=primary -o name | head -1); echo "$OLDP"
kubectl -n database exec $OLDP -c postgres -- psql -U postgres -d teslamate -tAc "select 'cars', count(*)::text from cars union all select 'drives', count(*)::text from drives union all select 'charging_processes', count(*)::text from charging_processes union all select 'positions', count(*)::text from positions union all select 'schema_migrations', max(version)::text from schema_migrations"
kubectl -n database exec $OLDP -c postgres -- psql -U postgres -d teslamate -tAc "select extname from pg_extension" | tr '\n' ' '; echo
```

Expected: five rows (positions will be the big one); note the numbers — they are compared again after the copy. Extensions typically `plpgsql cube earthdistance` (TeslaMate needs `cube`/`earthdistance`; `pg_restore` recreates them as superuser).

- [ ] **Step 2: Branch, remove, PR**

```bash
cd ~/repo/kichi-org/home-ops
git stash push -q -- .gitignore 2>/dev/null || true
git checkout -b chore/teslamate-cutover main
git rm -r -q kubernetes/apps/observability/teslamate
sed -i '' '/^  - \.\/teslamate\/ks\.yaml$/d' kubernetes/apps/observability/kustomization.yaml
grep -c teslamate kubernetes/apps/observability/kustomization.yaml || echo "reference removed"
git add kubernetes/apps/observability && git commit -q -m "chore(teslamate): move to the new cluster"
git push -u origin chore/teslamate-cutover
git checkout -q main && git stash pop -q 2>/dev/null || true
```

GitHub MCP `create_pull_request` (head `chore/teslamate-cutover`, base `main`), body: "Phase 3 — stops TeslaMate on the old cluster (its database stays). `v2` PR #PR_V2 enables it on talos-11 after the DB copy. Merge this one first." Record as `PR_MAIN`.

- [ ] **Step 3: Calvin merges `PR_MAIN`; confirm the old TeslaMate is gone**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
flux --namespace flux-system reconcile kustomization flux-system --with-source | tail -1
until ! kubectl -n observability get pods -l app.kubernetes.io/name=teslamate --no-headers 2>/dev/null | grep -q .; do sleep 10; done; echo "old teslamate stopped $(date +%T)"
kubectl -n observability get kustomization teslamate 2>&1 | tail -1
```

Expected: pod gone; Kustomization `teslamate` NotFound (pruned). Record the timestamp — this is the start of the data gap.

### Task 3: Copy the database old → new

**Files:** none (cluster operation, run from the new cluster's `postgres-1` pod).

**Interfaces:**

- Consumes: old primary `172.16.0.122:5432` (superuser creds from old `database/cloudnative-pg-secret`), new `postgres-rw` (superuser `postgres`, local trust inside the pod), TeslaMate role password from old `observability/teslamate-secret` key `DATABASE_PASS`.
- Produces: database `teslamate` owned by role `teslamate` on the new cluster with identical row counts.

- [ ] **Step 1: Create role + database on the new cluster (password taken from the old secret, never echoed)**

```bash
cd ~/repo/kichi-org/home-ops && export KUBECONFIG=$PWD/kubeconfig
TM_USER=$(kubectl -n observability get secret teslamate-secret -o jsonpath='{.data.DATABASE_USER}' | base64 -d)
TM_PASS=$(kubectl -n observability get secret teslamate-secret -o jsonpath='{.data.DATABASE_PASS}' | base64 -d)
OLD_SUPER_PASS=$(kubectl -n database get secret cloudnative-pg-secret -o jsonpath='{.data.password}' | base64 -d)
echo "user=$TM_USER pass_len=${#TM_PASS} old_super_len=${#OLD_SUPER_PASS}"
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl -n database exec -i postgres-1 -c postgres -- psql -U postgres -v ON_ERROR_STOP=1 -v u="$TM_USER" -v p="$TM_PASS" <<'EOF'
SELECT format('CREATE ROLE %I LOGIN PASSWORD %L', :'u', :'p') WHERE NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname = :'u') \gexec
SELECT format('ALTER ROLE %I WITH PASSWORD %L', :'u', :'p') \gexec
SELECT format('CREATE DATABASE %I OWNER %I', 'teslamate', :'u') WHERE NOT EXISTS (SELECT 1 FROM pg_database WHERE datname = 'teslamate') \gexec
EOF
kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -tAc "select datname, pg_get_userbyid(datdba) from pg_database where datname='teslamate'"
```

Expected: `user=teslamate pass_len=<n>`; final line `teslamate|teslamate`.

- [ ] **Step 2: Stream the dump from the old primary straight into the new database**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl -n database exec -i postgres-1 -c postgres -- bash -c "PGPASSWORD='$OLD_SUPER_PASS' pg_dump -h 172.16.0.122 -U postgres -d teslamate -Fc --no-password | pg_restore -U postgres -d teslamate --no-password --exit-on-error -j 2" 2>&1 | tail -5
echo "restore exit=${PIPESTATUS[0]}"
```

Expected: no output (or only `pg_restore: warning:` lines about extension comments) and `restore exit=0`. ~393 MB over 1 GbE — a minute or two. If `-j 2` fails with "parallel restore from stdin is not supported", drop `-j 2` (or dump to `/var/lib/postgresql/data/teslamate.dump` first, restore from the file, then `rm` it).

- [ ] **Step 3: Verify — counts, ownership, extensions, role can log in**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -d teslamate -tAc "select 'cars', count(*)::text from cars union all select 'drives', count(*)::text from drives union all select 'charging_processes', count(*)::text from charging_processes union all select 'positions', count(*)::text from positions union all select 'schema_migrations', max(version)::text from schema_migrations"
kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -d teslamate -tAc "select extname from pg_extension" | tr '\n' ' '; echo
kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -d teslamate -tAc "select count(*) from pg_tables where schemaname='public' and tableowner<>'teslamate'"
kubectl -n database exec postgres-1 -c postgres -- bash -c "PGPASSWORD='$TM_PASS' psql -h postgres-rw -U teslamate -d teslamate -tAc 'select current_user'"
kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -tAc "select pg_size_pretty(pg_database_size('teslamate'))"
```

Expected: the five numbers equal Task 2 Step 1 exactly (old TeslaMate was stopped before the dump); same extension list; `0` tables not owned by `teslamate` (if not 0: `REASSIGN OWNED BY postgres TO teslamate` inside the `teslamate` DB is **not** safe — instead run `ALTER TABLE ... OWNER TO teslamate` for the listed tables); `current_user = teslamate`; size ≈ 393 MB.

- [ ] **Step 4: Force a WAL switch so the migrated data is in R2 immediately**

```bash
kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -tAc "select pg_switch_wal()" >/dev/null && sleep 60 && kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -tAc "select last_archived_wal, coalesce(last_failed_wal,'-') from pg_stat_archiver"
```

Expected: `last_archived_wal` advanced, no failures. (The next 03:00 base backup will include the DB; optionally trigger one now with `kubectl -n database create -f - <<EOF … kind: Backup … EOF` using the same `method: plugin` block as `scheduledbackup.yaml`.)

### Task 4: Enable TeslaMate on the new cluster (merge `PR_V2`)

- [ ] **Step 1: Calvin merges `PR_V2`; reconcile and wait**

```bash
cd ~/repo/kichi-org/home-ops-v2 && export PATH=$HOME/.local/share/mise/shims:$PATH KUBECONFIG=$PWD/kubeconfig
git checkout -q v2 && git pull -q --ff-only && git branch -d feat/teslamate
flux --namespace flux-system reconcile kustomization flux-system --with-source | tail -1
until kubectl -n observability get kustomization teslamate -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}' 2>/dev/null | grep -q True; do sleep 15; done
kubectl -n observability rollout status deploy/teslamate --timeout=300s | tail -1
kubectl -n observability logs deploy/teslamate -c init-db --tail=5
kubectl -n observability logs deploy/teslamate -c app --since=5m | grep -iE 'migrat|Starting|error|logged in|online' | head -10
```

Expected: init container logs "Database … exists" / "Update User … Password"; app logs show Ecto migrations already up (no new migration applied), then the car appears (`… online` / `Start / :online`), no auth errors — the Tesla tokens live in the DB, encrypted with `ENCRYPTION_KEY` from the same 1Password item, so no re-login is needed.

- [ ] **Step 2: UI + DB liveness**

```bash
curl -sk --max-time 10 --resolve teslamate.kichi.live:443:172.16.0.31 -o /dev/null -w 'teslamate http=%{http_code}\n' https://teslamate.kichi.live/
sleep 120; kubectl -n database exec postgres-1 -c postgres -- psql -U postgres -d teslamate -tAc "select max(date) from positions"
```

Expected: `http=200`; `max(date)` is within the last few minutes if the car is awake, otherwise equal to the pre-cutover value (asleep cars log nothing — check `select state, start_date from states order by start_date desc limit 1`).

- [ ] **Step 3: Grafana**

```bash
curl -sk --max-time 10 --resolve grafana.kichi.live:443:172.16.0.31 'https://grafana.kichi.live/api/search?query=&type=dash-db' | jq -r '[.[] | select(.folderTitle=="TeslaMate")] | length'
DS=$(curl -sk --resolve grafana.kichi.live:443:172.16.0.31 'https://grafana.kichi.live/api/datasources' | jq -r '.[] | select(.name=="TeslaMate") | .uid')
curl -sk --resolve grafana.kichi.live:443:172.16.0.31 "https://grafana.kichi.live/api/datasources/uid/$DS/health" | jq -c '{status, message}'
```

Expected: `20` dashboards in folder TeslaMate (allow a few minutes for the operator to fetch them from GitHub); datasource health `{"status":"OK"}`. Then Calvin opens one of the TeslaMate dashboards (Overview / Drives) in a browser via `https://grafana.kichi.live` — note that name still resolves to the **old** Grafana until step 7; use a temporary `/etc/hosts` line `172.16.0.31 grafana.kichi.live` or the old Technitium record for the check.

### Task 5: Close out

- [ ] **Step 1: Execution log** — fill the section below on `v2` and `main` (`docs(plan): phase 3 execution log`), push both (background the push — it can stall).
- [ ] **Step 2: Memory** — `migration-progress`: phase 3 done (date, PRs, data-gap window), next = step 4 (*arr + qBittorrent + qui + Recyclarr).
- [ ] **Step 3: Rollback (only if needed)** — revert `PR_V2` (stops new TeslaMate), then revert `PR_MAIN` (old TeslaMate resumes on the old DB). Data logged on the new DB after the cutover would need a reverse copy — do the rollback decision within the first hour if at all.

---

## Execution log

- PR_V2 / PR_MAIN: _(Task 1 / Task 2)_
- Old counts (Task 2 Step 1) / new counts (Task 3 Step 3): _(…)_
- Data-gap window: old stopped _(time)_ → new online _(time)_
- Date completed: _(…)_
