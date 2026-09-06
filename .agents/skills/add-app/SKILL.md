---
name: add-app
description: Use when deploying a new application to the cluster — scaffolding a Flux Kustomization plus app-template HelmRelease under kubernetes/apps/ (new app, new service, "add X to the cluster")
---

# Add a New Application

Scaffolds `kubernetes/apps/<namespace>/<app>/` with a Flux Kustomization (`ks.yaml`) and an app-template HelmRelease. Every value below comes from current repo conventions — when in doubt, mirror a recent real app instead of inventing structure:

| Reference app                             | Shows                                                                     |
| ----------------------------------------- | ------------------------------------------------------------------------- |
| `kubernetes/apps/downloads/prowlarr`      | Standard web app: secrets, kopiur persistence, internal route                |
| `kubernetes/apps/downloads/qui`           | Same, plus the NFS `/data` mount from the NAS                             |
| `kubernetes/apps/downloads/recyclarr`     | Cronjob controller, config file via configMapGenerator                    |
| `kubernetes/apps/media/plex`              | Public route (`envoy-external`), LoadBalancer service, custom route rules |
| `kubernetes/apps/observability/teslamate` | CloudNativePG dependency, multi-item ExternalSecret                       |

Repo-wide rules that override anything below: no explanatory comments in manifests (rationale goes in the PR body), and show the proposed files before writing them when the user has not already approved the design.

## Step 1: Gather details

Ask the user (AskUserQuestion) for anything not already given:

1. **App name** and **namespace** (existing dirs: `ls kubernetes/apps/`)
2. **Image** repository + tag (upstream's current release)
3. **Port** the app listens on, and whether it gets a **route** (`<app>.kichi.live`); internal (`envoy-internal`, default) or public (`envoy-external` — only Plex is public today, via the Cloudflare tunnel)
4. **Persistence** — does the app store state? (→ kopiur backup component)
5. **Secrets** — env vars from 1Password? (→ ExternalSecret). Get the 1Password item name AND its exact field labels — never guess field names
6. **Config files** — mounted config? (→ configMapGenerator + `resources/`)
7. **Media/NAS access** — does it need `/data` from `kl-san-1`? (→ NFS persistence entry)
8. **Dependencies** — other Flux Kustomizations this app needs (e.g. `cloudnative-pg-cluster` in `database`)

## Step 2: Create the files

Layout:

```
kubernetes/apps/<namespace>/<app>/
├── ks.yaml
└── app/
    ├── kustomization.yaml
    ├── ocirepository.yaml
    ├── helmrelease.yaml
    ├── externalsecret.yaml      # only if secrets
    └── resources/               # only if config files
```

### ks.yaml

Keys under `spec` are alphabetical. `wait: false` is always present (repo convention).

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/kustomize.toolkit.fluxcd.io/kustomization_v1.json
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: <app>
spec:
  dependsOn:
    - name: onepassword
     namespace: external-secrets
  interval: 1h
  path: ./kubernetes/apps/<namespace>/<app>/app
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  targetNamespace: <namespace>
  wait: false
```

`dependsOn`: `onepassword`/`external-secrets` when there is an ExternalSecret, `openebs`/`openebs-system` when there is a PVC, plus any user-specified dependencies (same-namespace ones omit `namespace`). Omit `dependsOn` entirely if nothing applies. Do not add `commonMetadata`, `timeout`, `decryption` or `deletionPolicy` — `kubernetes/flux/cluster/ks.yaml` patches the defaults onto every child Kustomization.

**If the app has persistence**, add these to `spec` (alphabetical position — `components` first, `postBuild` after `path`). The component creates the PVC `<app>` on `openebs-hostpath` via a kopiur `Restore` populator and adds the `SnapshotPolicy` + hourly `SnapshotSchedule` to the R2 repository, so a fresh deploy and a disaster recovery are the same manifest. See `kubernetes/components/kopiur/backup/` for the knobs.

```yaml
components:
  - ../../../../components/kopiur/backup
postBuild:
  substitute:
    APP: <app>
```

Optional overrides under `substitute`, only when the defaults don't fit: `KOPIUR_CAPACITY` (default `1Gi`), `KOPIUR_CRON` (default `H * * * *`; Plex uses `H 4 * * *`), `KOPIUR_UID`/`KOPIUR_GID` (default `1000`; quote them, e.g. `"0"`). Include `postBuild.substitute.APP` whenever any component is used; omit `components`/`postBuild` entirely otherwise.

### app/kustomization.yaml

Resources are listed alphabetically.

```yaml
---
# yaml-language-server: $schema=https://json.schemastore.org/kustomization
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ./externalsecret.yaml # only if secrets
  - ./helmrelease.yaml
  - ./ocirepository.yaml
```

**If the app mounts config files**, put them in `resources/` and append:

```yaml
configMapGenerator:
  - name: <app>-configmap
   files:
     - config.yaml=./resources/config.yaml
generatorOptions:
  disableNameSuffixHash: true
```

If the config file itself contains `${...}` text, add `annotations: { kustomize.toolkit.fluxcd.io/substitute: disabled }` under `generatorOptions` so Flux leaves it alone.

### app/ocirepository.yaml

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/source.toolkit.fluxcd.io/ocirepository_v1.json
apiVersion: source.toolkit.fluxcd.io/v1
kind: OCIRepository
metadata:
  name: <app>
spec:
  interval: 15m
  layerSelector:
    mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip
    operation: copy
  ref:
    tag: <version>
  url: oci://ghcr.io/bjw-s-labs/helm/app-template
```

**Never hardcode `<version>` from memory** — use the version the rest of the repo is on:

```bash
grep -l app-template kubernetes/apps/*/*/app/ocirepository.yaml | xargs grep -h "tag:" | sort | uniq -c | sort -rn | head -1
```

### app/helmrelease.yaml

```yaml
---
# yaml-language-server: $schema=https://raw.githubusercontent.com/bjw-s-labs/helm-charts/main/charts/other/app-template/schemas/helmrelease-helm-v2.schema.json
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <app>
spec:
  chartRef:
    kind: OCIRepository
    name: <app>
  interval: 1h
  values:
    defaultPodOptions:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        fsGroupChangePolicy: OnRootMismatch
    controllers:
      <app>:
        annotations:
          reloader.stakater.com/auto: "true"
        containers:
          app:
            image:
              repository: <image-repo>
              tag: <image-tag>@sha256:<digest>
            env:
              TZ: Asia/Kuala_Lumpur
              <APP>__PORT: &port <port>
            probes:
              liveness: &probes
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: <health-path>
                    port: *port
                  initialDelaySeconds: 0
                  periodSeconds: 10
                  timeoutSeconds: 1
                  failureThreshold: 3
              readiness: *probes
            resources:
              requests:
                cpu: 10m
              limits:
                memory: 256Mi
            securityContext:
              allowPrivilegeEscalation: false
              readOnlyRootFilesystem: true
              capabilities: { drop: ["ALL"] }
    persistence:
      tmp:
        type: emptyDir
    service:
      app:
        ports:
          http:
            port: *port
```

Conventions baked into that block:

- **Pod security lives in `defaultPodOptions`**, not per-controller. Adjust `runAsUser`/`runAsGroup`/`fsGroup` to what the image requires; drop `runAsNonRoot` only if the image genuinely can't run non-root (technitium is the one example, and it overrides `KOPIUR_UID`/`GID` to match).
- **`&port` anchor**: declare it on whichever env var (or probe port) first carries the number, reuse it everywhere else. If the image has no port env var, anchor it on the probe's `port:` like Plex does.
- **Custom probes** pointing at the app's real health endpoint (`/ping` for *arr apps, `/health` for qui, `/identity` for Plex). If upstream has no health endpoint, use `liveness: { enabled: true }` / `readiness: { enabled: true }` instead.
- **Image tag is digest-pinned** (`<tag>@sha256:<digest>`). Take the digest from the upstream package page or `docker manifest inspect`; a plain tag is acceptable as a last resort because Renovate pins the digest on its next run.
- **`tmp` emptyDir** stays whenever `readOnlyRootFilesystem: true` is set.

**Optional value blocks** — keys under `values` follow `.agents/instructions/sorting.instructions.md`: `defaultPodOptions` first, then alphabetical (`controllers`, `persistence`, `route`, `service`).

Route (web UI/API):

```yaml
route:
  app:
    hostnames:
      - "{{ .Release.Name }}.kichi.live"
    parentRefs:
      - name: envoy-internal # envoy-external only for the public app
       namespace: network
```

Persistence (pairs with the kopiur component in ks.yaml; the claim name is always `<app>`):

```yaml
persistence:
  config:
    existingClaim: <app>
    globalMounts:
      - path: /config
```

NAS data (media, downloads):

```yaml
persistence:
  data:
    type: nfs
    server: kl-san-1.localdomain
    path: /volume1/data
    globalMounts:
      - path: /data
       readOnly: true # drop for writers (qbittorrent, *arr)
```

Config file mount (pairs with configMapGenerator):

```yaml
persistence:
  config-file:
    type: configMap
    name: "{{ .Release.Name }}-configmap"
    globalMounts:
      - path: /config/config.yaml
       subPath: config.yaml
```

Secrets: add to the container, after `env`:

```yaml
envFrom:
  - secretRef:
     name: "{{ .Release.Name }}-secret"
```

Cronjob workloads: copy the `type: cronjob` / `cronjob:` / `pod.restartPolicy: Never` block from recyclarr and skip `service`, `route` and probes.

### app/externalsecret.yaml (only if secrets)

```yaml
---
# yaml-language-server: $schema=https://kubernetes-schemas.pages.dev/external-secrets.io/externalsecret_v1.json
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: <app>
spec:
  secretStoreRef:
    kind: ClusterSecretStore
    name: onepassword
  target:
    name: <app>-secret
    template:
      data:
        SOME_ENV_VAR: "{{ .FIELD_LABEL }}"
  dataFrom:
    - extract:
       key: <1password-item>
```

Convention: `metadata.name` is `<app>`, the generated Secret is `<app>-secret`, `dataFrom.extract.key` is the 1Password item title, and `template.data` maps env vars to the item's field labels verbatim (`{{ .PROWLARR_API_KEY }}`). No `refreshInterval`, `creationPolicy` or `rewrite` — the repo doesn't use them. Pull from several items by listing several `extract` entries (teslamate). A wrong field label renders an empty value with no error; if the labels weren't provided and you can't ask, insert `<FIXME: 1password field label>` placeholders and call them out.

## Step 3: Register in the namespace kustomization

Add `./<app>/ks.yaml` to `kubernetes/apps/<namespace>/kustomization.yaml` `resources`, in alphabetical position among the app entries (`./namespace.yaml` stays first).

**New namespace?** Create `kubernetes/apps/<namespace>/` with `namespace.yaml` and `kustomization.yaml` copied from an existing namespace (e.g. `media`): keep `namespace: <namespace>`, the `../../components/alerts` component, and the literal `name: _` plus the `prune: disabled` annotation in namespace.yaml (kustomize renames it). Nothing else to register — `kubernetes/flux/cluster/ks.yaml` reconciles every directory under `kubernetes/apps/`.

## Step 4: Verify

```bash
kustomize build kubernetes/apps/<namespace>/<app>/app   # must render; ${APP} vars staying literal is expected
kustomize build kubernetes/apps/<namespace>                # ks.yaml + namespace kustomization render
```

Only `oxfmt` runs at commit (lefthook); the `flate` conformance run happens in CI on the PR. Show the user the created files and get confirmation before committing. Commit style: `feat(<app>): deploy <app>` — subject line only, no body, no trailers. Open a PR; Calvin merges manually and Flux applies it.

## Common mistakes

- **Copying a chart version or image tag from this skill or memory** — always read the current version from the repo (Step 2 command) and upstream.
- **Using volsync** — this repo migrated to kopiur; `components/volsync` no longer exists.
- **Setting `wait: true` on a leaf app, or adding `commonMetadata`/`timeout` to ks.yaml** — leaf apps keep the explicit `wait: false`; `wait: true` belongs only on Kustomizations that others `dependsOn` and that define no `healthChecks` (Flux ignores `healthChecks` when `wait` is true).
- **Putting `securityContext` under `controllers.<app>.pod`** — pod-level security goes in `defaultPodOptions`.
- **Forgetting `reloader.stakater.com/auto`** — without it, secret/config changes don't restart pods.
- **`readOnlyRootFilesystem: true` without a tmpfs** — apps that write to `/tmp` will crash; keep the `tmp` emptyDir.
- **Adding explanatory comments to the manifests** — rationale belongs in the PR body.
- **Skipping the sorting conventions** — HelmRelease values follow `.agents/instructions/sorting.instructions.md`.
- **Adding a CiliumNetworkPolicy by default** — no app in this repo uses one; only add if asked.
