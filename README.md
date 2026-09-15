# civitas-goat-addon

[CIVITAS/CORE](https://docs.core.civitasconnect.digital/) Ansible addon that installs the [GOAT](https://github.com/plan4better/goat) Helm chart and wires it to civitas's own Postgres (central-db), S3-compatible object storage (an in-cluster MinIO deployed by this addon by default, or any external S3 endpoint — see §External S3), and Keycloak realm.

## Status

Installs the GOAT Helm chart (`oci://ghcr.io/plan4better/charts/goat` v0.4.x) into civitas with:

- `<env>-goat-stack` namespace + civitas CA mirroring
- `goat` + `windmill` databases provisioned in civitas's central-db (Zalando `preparedDatabases` patch)
- Keycloak `goat-web` OIDC client in the civitas realm (idempotent)
- MinIO deployment + bucket + DuckLake catalog bootstrap (or wired to an existing S3 endpoint — see [External S3](#external-s3-compatible-object-storage))
- Helm install of GOAT (core, web, geoapi, processes, windmill server + 4 workers, redis)
- All GOAT images pinned to a single release tag (`inv_addons.goat.release`) — see [Versioning](#versioning)

## Routing

The whole stack is served from **one Ingress hostname** (`<subdomain>.<INGRESS_DOMAIN | default(DOMAIN)>`). Path prefixes route to services; the ingress class is `inv_k8s.ingress_class`, TLS is one shared secret (`<ingress-host>-tls`) issued by an explicit `Certificate` resource created by `tasks/06_certificate.yml`. Five Ingresses reference the same secret; owning the Certificate ourselves prevents cert-manager's ingress-shim from spawning five racing reconcilers.

The browser-facing URL (`web.public_url`, default `https://<subdomain>.<DOMAIN>`) drives `NEXT_PUBLIC_*`, `NEXTAUTH_URL`, `CLIENT_URL`, `web.auth.publicUrl`, Keycloak `redirectUris`/`webOrigins`/`postLogoutRedirectUris`, and `S3_PUBLIC_ENDPOINT_URL`. When there is no fronting reverse proxy, `DOMAIN == INGRESS_DOMAIN` and the public URL host equals the ingress host. On the `/<bucket>` S3 Ingress (and only there), `nginx.ingress.kubernetes.io/upstream-vhost` rewrites the `Host` header the upstream sees to the browser-facing name — SigV4 signs the request host, so presigned S3 URLs would otherwise fail validation behind a fronting reverse proxy. The chart's four service Ingresses set no `upstream-vhost`; a fronting proxy is expected to forward the original `Host`/`X-Forwarded-Host` itself (Next.js Server Actions validate Origin against these — symptom when misconfigured: "Invalid Server Actions request").

| Path prefix     | Service          | Mechanism                                                                                |
|-----------------|------------------|------------------------------------------------------------------------------------------|
| `/`             | goat-web         | host root, no rewrite (Next.js `basePath` is build-time)                                 |
| `/core`         | goat-core        | `API_V2_STR="/core/api/v2"` carries the prefix, no rewrite                               |
| `/geoapi`       | geoapi           | prefix stripped via ingress profile (nginx: `rewrite-target`; traefik: `stripPrefix` MW) |
| `/processes`    | processes        | prefix stripped via ingress profile (nginx: `rewrite-target`; traefik: `stripPrefix` MW) |
| `/<bucket>`     | MinIO            | path prefix equals bucket name (SigV4 constraint) — no rewrite                           |
| —               | windmill         | no public ingress; `processes` reaches it in-cluster                                     |

### Supported ingress controllers

Selected by `inv_k8s.ingress_class`; a startup `assert` fails cleanly if the value has no matching profile.

| `inv_k8s.ingress_class` | Prefix-strip for geoapi/processes | Extra operator setup |
|---|---|---|
| `nginx`   | `nginx.ingress.kubernetes.io/rewrite-target` annotation | none |
| `traefik` | Per-service `stripPrefix` `Middleware` CRD (`tasks/04_ingress_middleware.yml`) | HTTPS-redirect must be configured at the Traefik entrypoint (or via a `redirectScheme` middleware) — the `nginx.ingress.kubernetes.io/ssl-redirect` annotation is silently ignored on Traefik |

Traefik Middleware API group defaults to `traefik.io/v1alpha1` (Traefik v3). For Traefik v2 clusters, override:

```yaml
inv_addons:
  goat:
    ingress:
      traefik_api_group: "traefik.containo.us/v1alpha1"
```

Adding a third controller (e.g. APISIX) is additive: add a profile block under `goat_addon.ingress.profiles.<class>` in `vars/default.yml`, wire whatever per-service CRDs it needs, and the assert stops failing. Do not add elif branches in the template.

### Strategic note — this abstraction is temporary

The profile map exists **only** because geoapi and processes do not support FastAPI's `root_path`. If the upstream `ROOT_PATH` change lands in `plan4better/goat`, both services can serve under their own prefix like goat-core already does — at which point the profile map, the Middleware CRDs, and every controller-conditional annotation can be deleted in favour of plain `pathType: Prefix` paths with no annotations on any controller. Treat any additions to this layer as debt.

**Accepted trade-offs.** geoapi/processes emit OGC/HATEOAS/TileJSON absolute URLs from `request.base_url` that omit the `/geoapi` or `/processes` prefix. The GOAT UI does not consume those URLs (it composes every backend URL itself from `NEXT_PUBLIC_*`), so this only affects external clients pointing directly at those endpoints. Similarly, `/geoapi/api/docs` and `/processes/api/docs` cannot fetch their spec (`openapi_url` is hardcoded to `/api/openapi.json`), though the raw JSON stays reachable.

## Versioning

Every image built from the `plan4better/goat` monorepo pins to a single release tag. Set it via inventory:

```yaml
inv_addons:
  goat:
    release: "v2.4.60"   # applies to core / web / geoapi / processes /
                         # windmill-server / windmill-worker-{default,tools,print}
```

**Why they move together.** `geoapi`, `processes` and `windmill-worker-tools` share a DuckLake catalog through a Postgres schema; the DuckDB extension baked into each image writes the catalog's on-disk format, so a version skew across services produces `DuckLake catalog version mismatch` at attach time. `core` does not use DuckDB (delegates DuckLake to geoapi over HTTP), but is pinned for API compatibility. Non-GOAT images (`redis`, `minio`, `minio_mc`) are pinned independently in `vars/software_references.yml`.

The DuckLake bootstrap Job (`tasks/05_minio.yml`) runs from the `processes` image and uses `AUTOMATIC_MIGRATION TRUE` on ATTACH, so it can upgrade an older catalog to the current release's format. This is the *only* place a catalog upgrade can happen — goatlib's runtime attach hardcodes its option list.

## Inventory

Full schema for the operator's `cc_cli_inventory.yml` under `inv_addons.goat`:

```yaml
inv_addons:
  goat:
    enable: true
    namespace: "{{ ENVIRONMENT }}-goat-stack"
    release: "v2.4.60"
    chart:
      ref: "oci://ghcr.io/plan4better/charts/goat"
      version: "0.4.1"
    db:
      # Optional — Postgres database names. Defaults shown.
      # Override only if you need to co-tenant multiple goat installs in
      # one Postgres cluster. NB windmill's *role* names (windmill_admin,
      # windmill_user) are hardcoded by its migrations regardless.
      goat_db_name: "goat"
      windmill_db_name: "windmill"
      # Optional — runtime Postgres endpoint that the workloads dial.
      # Defaults to the Zalando master service. DB provisioning is
      # unaffected (01_db.yml drives Zalando by cluster name, not
      # endpoint).
      # host: "central-db.{{ inv_central_db.ns_name }}.svc.cluster.local"
      # port: 5432
      # Per-service overrides, default to `host`/`port`.
      # windmill_host: "central-db.{{ inv_central_db.ns_name }}.svc.cluster.local"
      # windmill_port: 5432
      # geoapi_host: "central-db.{{ inv_central_db.ns_name }}.svc.cluster.local"
      # geoapi_port: 5432
    keycloak:
      client_id: "goat-web"
    ingress:
      # Traefik v2 clusters only — see "Supported ingress controllers".
      traefik_api_group: "traefik.io/v1alpha1"
    web:
      # Base subdomain. Composed with DOMAIN / INGRESS_DOMAIN to yield the
      # ingress host and the public URL — see "Routing".
      subdomain: "goat"
      # Optional override for the browser-facing URL. Leave unset to
      # accept the default `https://<subdomain>.<DOMAIN>`. Set explicitly
      # only for dev setups where the browser URL is not derived from
      # subdomain + DOMAIN.
      # public_url: "https://goat.{{ DOMAIN }}"
    s3:
      # Bucket name. Also the path prefix on the in-cluster MinIO ingress:
      # presigned URLs are path-style, so any deviation from `/<bucket>`
      # breaks SigV4 signatures.
      bucket: "goat"
      # Deployment mode. `in_cluster` (default) deploys the addon's own
      # single-replica MinIO. `external` points at any pre-existing
      # S3-compatible endpoint (external MinIO, R2, Wasabi, B2, AWS S3).
      # See §External S3.
      mode: "in_cluster"
      # External-mode endpoint — required when mode: external. Only the
      # cluster-internal URL is configured; the browser-facing S3 URL is
      # always the public GOAT URL in both modes (see §External S3).
      endpoint: ""            # http(s)://... cluster-visible (in-pod code)
      # External-mode credentials — required when mode: external.
      # Keep in ansible-vault; the addon writes them into a
      # `goat-s3-credentials` Secret in the goat namespace.
      access_key: ""
      secret_key: ""
      # Runtime S3 settings — used in both modes.
      region: "us-east-1"
      provider: "minio"       # or "aws" — passed through as S3_PROVIDER
      force_path_style: true
      # Tunables for the `/<bucket>` passthrough Ingress (both modes).
      proxy:
        # Upload size cap — ingress-nginx defaults to 1 MB, which 413s
        # any real dataset upload.
        max_body_size: "5g"
        # Upstream TLS verification. No-op until a `proxy-ssl-secret`
        # (client-CA Secret) is provisioned — kept as a plumbing hook.
        ssl_verify: false
        # SNI for the upstream TLS handshake. Empty → derived from the
        # `endpoint` hostname.
        ssl_name: ""
        # `Host` header sent to the S3 upstream. Unset → browser-facing
        # host (SigV4 signs it); "" → omit the annotation (pass-through);
        # any other string → literal override.
        # upstream_vhost: ""
```

### Inventory key vs. `helm_values`

Two override surfaces exist, with a firm boundary:

- **First-class inventory key** (`inv_addons.goat.*`) — only for values the addon *tasks* consume: ingress hosts and TLS secrets, DB wiring, the S3 endpoint/ExternalName/annotations, Keycloak client. These values shape resources outside the Helm release, so they must flow through the addon's derivations.
- **`inv_addons.goat.helm_values`** — pure chart tuning (replicas, resources, extra service env, …). Recursively merged over the rendered values file in `tasks/03_helm_install.yml`, so any chart key can be overridden without a fork.

**Caveat:** `helm_values` overrides only the *chart* values — never the task-managed resources derived from inventory keys. Cross-cutting values (S3 endpoint, subdomain, DB host) must NOT be set via `helm_values`: the chart would see the override but the MinIO Ingress, ExternalName Service, Certificate and Keycloak client would keep the inventory-derived values and silently diverge.

S3 backends without POST-Object support (e.g. Dell ECS) need GOAT images that upload via presigned PUT; override the image source per service through the `software.addon_goat.images.<svc>` registry overrides in your inventory (see "Image references").

### External S3-compatible object storage

Set `inv_addons.goat.s3.mode: external` to skip the addon's in-cluster MinIO Deployment/Service and point every S3-consuming service (goat-core, geoapi, processes, both windmill workers, DuckLake bootstrap) at a pre-existing endpoint. Works for any S3-compatible service: MinIO elsewhere in the cluster, Cloudflare R2, Wasabi, Backblaze B2, or real AWS S3.

```yaml
inv_addons:
  goat:
    s3:
      bucket: "goat"
      mode: "external"
      endpoint: "https://s3.eu-central-1.amazonaws.com"
      region: "eu-central-1"
      provider: "aws"
      force_path_style: false
      # Keep the two values in ansible-vault under the `secrets` tree.
      access_key: "{{ secrets.inv_addons.goat.s3_access_key }}"
      secret_key: "{{ secrets.inv_addons.goat.s3_secret_key }}"
```

**Browser-facing endpoint = public GOAT URL, always.** There is no `public_endpoint` variable to set. In both modes the addon deploys a `/<bucket>` Ingress on the public GOAT host; presigned URLs returned by goat-core embed that URL as `S3_PUBLIC_ENDPOINT_URL`. In `in_cluster` mode the Ingress backend is the addon-deployed MinIO Service; in `external` mode it is a namespace-local `ExternalName` Service (`goat-s3-upstream`) that resolves to `endpoint`. This keeps the browser origin equal to the GOAT origin — no second host to cover with a TLS cert, no cross-origin CORS surface.

Preconditions (all asserted at addon start when `mode: external`):

1. **`access_key` + `secret_key` supplied** via inventory (source them from ansible-vault under `secrets.inv_addons.goat.s3_access_key` / `s3_secret_key`). The addon materializes them into a `goat-s3-credentials` Secret in the goat namespace on every run — no manual `kubectl create secret` step and no cross-namespace secret references. If your credentials live in a Secret elsewhere, copy the values into the vault once; the addon owns the in-namespace Secret from then on.
2. **Bucket** identified by `s3.bucket` **already exists** at the endpoint. The addon skips its `goat-bucket-init` Job in external mode — external credentials are typically scoped to a specific bucket without `s3:CreateBucket`, and the bucket is provisioned by the operator (Terraform, cloud console, `mc mb`, etc.) before the playbook runs. The credentials must have `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, and `s3:ListBucket` on this bucket — DuckLake-init writes catalog metadata to it on every run.
3. **`endpoint`** is reachable from the goat namespace (in-cluster DNS or public network). Used by boto3 / DuckDB / `mc` inside pods, and by the `goat-s3-upstream` `ExternalName` Service (a plain DNS alias — `endpoint` must be a hostname, not an IP literal). Operator-side firewall rules must permit egress from the goat namespace to the storage host; the addon does not test reachability.
4. **`region`** matches how SigV4 is signed on the target. AWS S3 requires the bucket's actual region; MinIO / R2 / etc. usually accept any value but should match what the endpoint advertises.
5. **`force_path_style`**: `true` (default) works for MinIO / R2 / Wasabi / Backblaze B2. Real AWS S3 has been deprecating path-style since 2020 and buckets created in some regions after that date only accept virtual-hosted-style — set `false`.
6. **`provider`**: passed through as `S3_PROVIDER` to goat-core / geoapi / processes / windmill workers. Default `minio` matches historical behavior; set `aws` for real AWS S3 if goat-core's config.py branches on it.
7. **CORS at the storage backend.** Because the browser talks to the S3 via the `/<bucket>` path on the public GOAT host, the presigned request looks (to the pod that ultimately handles it) like `Host: <public GOAT host>` — this is set explicitly through `nginx.ingress.kubernetes.io/upstream-vhost` so SigV4 validates. If the external S3 enforces a CORS allow-list on the request `Origin`, the operator must add the public GOAT origin (`https://<subdomain>.<DOMAIN>`) to it. Symptom when missing: uploads fail with an opaque CORS error against a healthy backend. Out of scope for the addon.

**DuckLake catalog binding.** The DuckLake catalog metadata (Postgres `ducklake.*` tables) stores the S3 endpoint written at bootstrap. Switching `endpoint` on an existing install leaves the catalog pointing at the old location; the addon's DuckLake init sets `AUTOMATIC_MIGRATION TRUE` on ATTACH which handles catalog *format* upgrades, but not endpoint relocation. Treat endpoint changes as a re-bootstrap.

**Cost of leftover state.** Switching from `in_cluster` → `external` does not delete the in-cluster MinIO Deployment/PVC/Service/Ingress if they already exist from a prior run. Delete manually (`kubectl -n <env>-goat-stack delete deploy,svc,ingress,pvc minio minio-data`) to reclaim the disk.

## How to use

In your civitas-core fork:

```sh
# Civitas-core's .gitignore excludes `core_platform/addons/`, so the addon
# is cloned directly (not added as a tracked submodule). The Ansible
# playbook reads files from the path regardless of git state.
git clone https://github.com/plan4better/civitas-goat-addon.git \
          core_platform/addons/goat_addon
```

Register the addon's `tasks.yml` in inventory:

```yaml
inv_addons:
  import: true
  addons:
    - "addons/goat_addon/tasks.yml"
  goat:
    # ...see Inventory schema above...
```

Run the civitas playbook scoped to addons:

```sh
ansible-playbook -i cc_cli_inventory.yml core_platform/playbook.yml \
  --tags "addons"
```

## Compatibility

| addon | civitas-core | GOAT chart | GOAT release |
|---|---|---|---|
| **current** | v1.7.x+ | v0.4.x | v2.4.60 |
| v0.2.0      | v1.5.x+ | v0.3.x | mixed (`latest` / `f59d1e3` / `v2.4.36`) |
| v0.1.x      | v1.5.x+ | v0.1.x |  |

Addon version is independent of civitas-core's — tag the addon on its own semver.

## Image references (mirror-friendly)

Every container image this addon spins up — both chart-deployed services and addon-deployed ones (MinIO, mc) — is enumerated in `vars/software_references.yml` under `software.addon_goat.images.*`. Override `registry:` per image in your inventory to point at a private mirror.

```sh
yq '.software.addon_goat.images[] | "\(.registry)/\(.repository):\(.tag)"' \
   vars/software_references.yml
```

**Caveat.** Civitas-core's own `tools/extract-images` and `tools/harbor` playbooks currently only enumerate images under a top-level `software:` key, whereas this addon uses `software.addon_goat:`. As a result the addon's images are not picked up by that tooling today. Track this as a known integration gap.

## Known caveats

- **`subdomain` and `web.public_url` are load-bearing across the whole stack**: `inv_addons.goat.web.subdomain` composes the ingress-visible hostname (`<subdomain>.<INGRESS_DOMAIN>`) and the browser-facing URL (`https://<subdomain>.<DOMAIN>`); `web.public_url` optionally overrides the latter. Together they drive every `NEXT_PUBLIC_*` env baked into the goat-web bundle, `CLIENT_URL` on goat-core, the Keycloak `redirectUris` / `webOrigins` / `postLogoutRedirectUris`, and `S3_PUBLIC_ENDPOINT_URL` (for in-cluster MinIO). The ingress name must resolve to the ingress load balancer — DNS record and cluster-issuer both have to agree. Changing subdomain or domain rotates the addon into a new host-derived TLS secret (`<ingress-host>-tls`); the OLD secret lingers in the namespace after the change and should be deleted manually to reclaim quota. The Keycloak client is PUT on every run against the same URL template, so hostname changes now propagate to `redirectUris` immediately — no more `invalid parameter: redirect_uri` after a rename.

- **The `cacert` mount and CA env vars are conditional on `inv_k8s.ingress.ca_path`**: the `NODE_EXTRA_CA_CERTS` (web) and `REQUESTS_CA_BUNDLE` / `SSL_CERT_FILE` (geoapi, processes) env vars, together with the `cacert` `extraVolumes` / `extraVolumeMounts` on those three services, are only rendered when `inv_k8s.ingress.ca_path` is set — the same condition that gates the `cacert` ConfigMap in `tasks/00_namespace.yml`. On clusters that use an ACME-issued certificate (chain trusted by the Node / OpenSSL default store), leave `ca_path` unset and no bundle is mounted. Previously the env vars were always emitted and goat-web logged `Warning: Ignoring extra certs from /etc/ssl/cacert/cacert.crt … No such file or directory` at every start. Do NOT substitute `kube-root-ca.crt`: that ConfigMap signs `kubernetes.default.svc`, not the public ingress cert, and is useless as a trust anchor for outbound HTTPS.

- **`goat-web` server-side fetches hairpin through the ingress**: the Next.js server calls `NEXT_PUBLIC_API_URL` from inside the pod (SSR of the org-creation flow), which routes back out through the same ingress that serves the browser. Until a valid cert is served on the ingress host, Node rejects the chain with `SELF_SIGNED_CERT_IN_CHAIN` and the UI silently loops on the organization-creation screen — no error in the browser console, no HTTP error visible to the user, just an endless refresh (the POST to `/organizations` on the second attempt then answers `{"detail":"User has already an organization"}` even though the frontend never got the first response). Confirm the cert served on the ingress host is trusted by the goat-web pod (`NODE_EXTRA_CA_CERTS` mounts the civitas CA bundle from `inv_k8s.ingress.ca_path` — set that in inventory when the ingress cert chains up through a private CA; a real ACME cert needs no extra bundle).

- **Postgres-operator stale-password**: if civitas's Zalando postgres-operator has lost its in-memory state (K8s secret rotated but DB password unchanged), `01_db.yml` will hang waiting for the goat secret to appear. Restart the operator pod (`kubectl -n <operator-ns> delete pod -l app.kubernetes.io/name=postgres-operator`) and retry.

- **`storage_class` is a dict, not a string**: civitas-core's `inv_k8s.storage_class` is `{loc, rwo, rwx}`. The chart wants a single string; the rendered values file picks `loc` by default. Override `inv_k8s.storage_class.loc` (or edit `templates/goat_values.yml` in a fork) for multi-node setups that need `rwx` storage.

- **`local-path` for the DuckLake PVC is single-node ONLY**: `local-path` is not a CSI driver and does not reject `ReadWriteMany` at bind time — it silently binds RWX on any node, and each node then gets a *separate empty directory*. On multi-node clusters this means `layer_import` (worker-tools) writes parquet files that `geoapi` cannot see, with no error anywhere: the tile requests just return empty results. Use a real RWX class (CephFS / NFS / EFS) on multi-node, or pin the data-touching pods to a single node. The addon accepts `inv_addons.goat.ducklake.storage_class` as an override so operators can point at a filesystem-backed class without moving `inv_k8s.storage_class.rwx` globally — block-backed classes (Ceph RBD, EBS, GCE PD, Azure Disk) cannot satisfy RWX and the PVC will sit Pending indefinitely with only a generic "waiting for external provisioner" event.

- **`goat-core` has no curl**: the image is stripped to essentials. Don't write tasks that `kubernetes.core.k8s_exec` shell-out into the container; rely on Kubernetes probes instead.

- **Ansible Jinja2 in dict keys**: if you ever need to add a third `preparedDatabases` entry with a name derived from a variable, note that Ansible does NOT evaluate Jinja inside YAML dict keys. Workaround in `01_db.yml`: a `set_fact` + a single Jinja dict-literal expression.

- **Upgrading from a version that used `minio-credentials` / `minio-bucket-init`**: the previous names were MinIO-branded, misleading once external non-MinIO backends were supported. Renames:
  - Secret `minio-credentials` → `goat-s3-credentials`
  - Job `minio-bucket-init` → `goat-bucket-init` (in-cluster mode only — the bucket-init Job no longer runs in external mode, since external credentials typically lack `s3:CreateBucket` and the bucket is provisioned out-of-band by the operator)

  Upgrade steps:
  - **External S3 mode**: no data risk. Delete the leftover resources to keep the namespace tidy:
    ```sh
    kubectl -n <env>-goat-stack delete secret minio-credentials --ignore-not-found
    kubectl -n <env>-goat-stack delete job minio-bucket-init --ignore-not-found
    ```
  - **In-cluster MinIO mode**: the addon will *not* find the old credentials Secret under the new name and will generate fresh root credentials, which MinIO's on-disk data rejects at startup — bucket-init and DuckLake-init then fail. Before upgrading, copy the old Secret to the new name so the addon reuses the existing creds:
    ```sh
    kubectl -n <env>-goat-stack get secret minio-credentials -o yaml \
      | sed 's/name: minio-credentials/name: goat-s3-credentials/' \
      | kubectl apply -f -
    kubectl -n <env>-goat-stack delete secret minio-credentials
    kubectl -n <env>-goat-stack delete job minio-bucket-init --ignore-not-found
    ```

## License

[EUPL-1.2](LICENSE) — same as civitas-core.
