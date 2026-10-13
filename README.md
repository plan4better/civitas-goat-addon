# civitas-goat-addon

[CIVITAS/CORE](https://docs.core.civitasconnect.digital/) Ansible addon that installs the [GOAT](https://github.com/plan4better/goat) Helm chart and wires it to civitas's own Postgres (central-db), S3-compatible object storage (the GOAT chart's single-node Garage by default, or any attached S3 endpoint — see §External S3), and Keycloak realm.

## Status

Installs the GOAT Helm chart (`oci://ghcr.io/plan4better/charts/goat` v0.7.x, GOAT v3.0.9+; values and version history in the [chart README](https://github.com/plan4better/goat/blob/main/deploy/helm/goat/README.md)) into civitas with:

- `<env>-goat-stack` namespace + civitas CA mirroring
- `goat` + `windmill` databases provisioned in civitas's central-db (Zalando `preparedDatabases` patch)
- Keycloak `goat-web` OIDC client in the civitas realm (idempotent)
- Object storage: the chart's Garage with its buckets (or an existing S3 endpoint — see [External S3](#external-s3-compatible-object-storage)) and the shared data volume
- Helm install of GOAT (core, web, geoapi, processes, catalog, windmill server + 4 workers, redis), authenticated against the civitas realm — see [Authentication](#authentication)
- Preflight of an attached S3 bucket and its credentials — see [External S3](#external-s3-compatible-object-storage)
- Optional: GOAT's queries through central-db's connection pooler ([`db.pooler`](#connection-pooler-dbpooler)) and a public Windmill UI ([`windmill.ingress`](#windmill-ui-windmillingress))
- All GOAT images pinned to a single release tag (`inv_addons.goat.release`) — see [Versioning](#versioning)

## Routing

The whole stack is served from **one Ingress hostname** (`<subdomain>.<INGRESS_DOMAIN | default(DOMAIN)>`). Path prefixes route to services; the ingress class is `inv_k8s.ingress_class`, TLS is one shared secret (`<ingress-host>-tls`) issued by an explicit `Certificate` resource created by `tasks/certificate.yml`. Five Ingresses reference the same secret; owning the Certificate ourselves prevents cert-manager's ingress-shim from spawning five racing reconcilers.

**Path-style S3 is structural, not an operator choice.** The addon fronts S3 at `/<bucket>` on the public GOAT host, and SigV4 signs the canonical URI. Virtual-hosted addressing would sign a `<bucket>.<goat-host>` name that has no DNS record, TLS SAN, or Ingress rule behind it — the browser would immediately fail on DNS. Every supported backend is therefore addressed path-style; there is no operator knob because there is nothing to choose. See [§Supported backends](#supported-backends) for what this rules in and out.

The browser-facing URL (`web.public_url`, default `https://<subdomain>.<DOMAIN>`) drives `NEXT_PUBLIC_*`, `NEXTAUTH_URL`, `CLIENT_URL`, `web.auth.publicUrl`, Keycloak `redirectUris`/`webOrigins`/`postLogoutRedirectUris`, and `S3_PUBLIC_ENDPOINT_URL`. When there is no fronting reverse proxy, `DOMAIN == INGRESS_DOMAIN` and the public URL host equals the ingress host. On the `/<bucket>` S3 Ingress (and only there), `nginx.ingress.kubernetes.io/upstream-vhost` rewrites the `Host` header the upstream sees to the browser-facing name — SigV4 signs the request host, so presigned S3 URLs would otherwise fail validation behind a fronting reverse proxy. The chart's four service Ingresses set no `upstream-vhost`; a fronting proxy is expected to forward the original `Host`/`X-Forwarded-Host` itself (Next.js Server Actions validate Origin against these — symptom when misconfigured: "Invalid Server Actions request").

| Path prefix     | Service          | Mechanism                                                                                |
|-----------------|------------------|------------------------------------------------------------------------------------------|
| `/`             | goat-web         | host root, no rewrite (Next.js `basePath` is build-time)                                 |
| `/core`         | goat-core        | `API_V2_STR="/core/api/v2"` carries the prefix, no rewrite                               |
| `/geoapi`       | geoapi           | `ROOT_PATH=/geoapi`, no rewrite                                                          |
| `/processes`    | processes        | `ROOT_PATH=/processes`, no rewrite                                                       |
| `/catalog-api`  | catalog          | `ROOT_PATH=/catalog-api`, no rewrite (STAC API at `/catalog-api/stac`); not `/catalog`, which is goat-web's own catalog page |
| `/<bucket>`     | S3 gateway       | path prefix equals bucket name (SigV4 constraint) — no rewrite                           |
| `/goat-assets`  | Garage website   | provision mode, ingress-nginx: uploaded images, prefix stripped — see [Uploaded images](#uploaded-images-goat-assets) |
| —               | windmill         | no public ingress by default; `processes` reaches it in-cluster. Optional own host, see [Windmill UI](#windmill-ui-windmillingress) |

### Ingress controllers

Every service serves under its own prefix (`API_V2_STR` for core, `ROOT_PATH` for geoapi, processes and catalog, GOAT v3.0.3+), so all service Ingresses are plain `pathType: Prefix` paths with no rewrite annotations or middleware, on any controller. The one exception is provision mode's `/goat-assets` Ingress (see [Uploaded images](#uploaded-images-goat-assets)). The services also build their OGC/STAC `self`/`next` links and API docs with the prefix, so external clients (QGIS, STAC browsers) can follow them.

On Traefik, the `nginx.ingress.kubernetes.io/*` annotations are not emitted; configure the HTTPS redirect at the Traefik entrypoint (or via a `redirectScheme` middleware).

## Versioning

Every image built from the `plan4better/goat` monorepo pins to a single release tag. Set it via inventory:

```yaml
inv_addons:
  goat:
    release: "v3.0.9"    # applies to core / web / geoapi / processes / catalog /
                         # windmill-server / windmill-worker-{default,tools,print}
```

**Why they move together.** `geoapi`, `processes` and `windmill-worker-tools` share a DuckLake catalog through a Postgres schema; the DuckDB extension baked into each image writes the catalog's on-disk format, so a version skew across services produces `DuckLake catalog version mismatch` at attach time. `core` does not use DuckDB (delegates DuckLake to geoapi over HTTP), but is pinned for API compatibility. Non-GOAT images (`redis`, `garage`) are pinned independently in `vars/software_references.yml`.

Use release tags (`v3.0.9`). A commit tag that YAML reads as a number, such as `6225e97` (6225 × 10⁹⁷), reaches Helm as a number, and the upgrade stops at the chart's schema check (`got number, want string`): civitas-core's Helm step writes the values without quotes.

The DuckLake catalog is created by the chart: an init container on geoapi and processes runs goatlib's idempotent bootstrap from the same image the services run, so the catalog format always matches. A catalog *format* upgrade between DuckLake versions is one-way and is not done automatically; follow the GOAT release notes when one is announced.

## Inventory

Full schema for the operator's `cc_cli_inventory.yml` under `inv_addons.goat`:

```yaml
inv_addons:
  goat:
    enable: true
    namespace: "{{ ENVIRONMENT }}-goat-stack"
    release: "v3.0.9"
    chart:
      # OCI refs are installed with --version; a local .tgz path pins itself.
      ref: "oci://ghcr.io/plan4better/charts/goat"
      version: "0.7.0"
    db:
      # Optional — Postgres database names. Defaults shown.
      # Override only if you need to co-tenant multiple goat installs in
      # one Postgres cluster. NB windmill's *role* names (windmill_admin,
      # windmill_user) are hardcoded by its migrations regardless.
      goat_db_name: "goat"
      windmill_db_name: "windmill"
      # Optional — runtime Postgres endpoint that the workloads dial.
      # Defaults to the Zalando master service. DB provisioning is
      # unaffected (db.yml drives Zalando by cluster name, not
      # endpoint).
      # host: "central-db.{{ inv_central_db.ns_name }}.svc.cluster.local"
      # port: 5432
      # Per-service overrides, default to `host`/`port`.
      # windmill_host: "central-db.{{ inv_central_db.ns_name }}.svc.cluster.local"
      # windmill_port: 5432
      # geoapi_host: "central-db.{{ inv_central_db.ns_name }}.svc.cluster.local"
      # geoapi_port: 5432
      # Optional, not needed: GOAT's queries through a transaction pooler,
      # DuckLake and Windmill stay on the database. See §Connection pooler.
      # Default: off.
      pooler:
        enable: false
        # A PgBouncer of your own (required when `host` above is not
        # central-db); it must listen on `port`. Empty = central-db's pooler.
        # host: "pgbouncer.example.svc.cluster.local"
    keycloak:
      client_id: "goat-web"
      # Clients whose tokens GOAT's APIs accept. Default: client_id only.
      # See §Organizations and API tokens.
      allowed_clients: ["goat-web"]
      # Set false once your organization exists. Default: true.
      organization_creation: true
      # Optional login gate — see §Restricting login. Default: off.
      limit_access: false
      gate_role: "goat-user"
      gate_group: "goat-users"
      flow_alias: "goat-browser"
      deny_message: "Ihr Benutzerkonto ist nicht für GOAT freigeschaltet. Bitte wenden Sie sich an Ihre Administration."
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
      # Bucket name. Also the path prefix on the `/<bucket>` S3 gateway
      # ingress: presigned URLs are path-style, so any deviation from
      # `/<bucket>` breaks SigV4 signatures.
      bucket: "goat"
      # Deployment mode. `provision` (default) runs the GOAT chart's
      # single-node Garage (not HA). `attach` points at any pre-existing
      # S3-compatible endpoint (MinIO, Dell ECS, ...). See §External S3.
      mode: "provision"
      # Attach-mode endpoint — required when mode: attach. Only the
      # cluster-internal URL is configured; the browser-facing S3 URL is
      # always the public GOAT URL in both modes (see §External S3).
      endpoint: ""            # http(s)://... cluster-visible (in-pod code)
      # Attach-mode credentials — required when mode: attach.
      # Keep in ansible-vault; the addon writes them into a
      # `goat-s3-credentials` Secret in the goat namespace.
      access_key: ""
      secret_key: ""
      # Attach mode: the region the endpoint signs for. Provision mode
      # always uses Garage's `garage`.
      region: "us-east-1"
      # Provision mode: size of Garage's data volume (uploads, exports,
      # prints, images).
      pvc_size: "10Gi"
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
    windmill:
      # Optional public Windmill UI on its own host. See §Windmill UI.
      ingress:
        enable: false
        subdomain: "goat-windmill"       # default: <web.subdomain>-windmill
        # public_url: "https://goat-windmill.{{ DOMAIN }}"
    base_data:
      # Default source of the routing base data (GOAT_BASE_DATA_URL on the
      # tools worker, read from GOAT v3.0.5). Empty: GOAT's public server.
      source_url: ""
```

### Inventory key vs. `helm_values`

Two override surfaces exist, with a firm boundary:

- **First-class inventory key** (`inv_addons.goat.*`) — only for values the addon *tasks* consume: ingress hosts and TLS secrets, DB wiring, the S3 endpoint/ExternalName/annotations, Keycloak client. These values shape resources outside the Helm release, so they must flow through the addon's derivations.
- **`inv_addons.goat.helm_values`** — pure chart tuning (replicas, resources, extra service env, …). Recursively merged over the rendered values file in `tasks/helm_install.yml`, so any chart key can be overridden without a fork.

**Resources.** The addon sets no `resources`: every service runs with the chart's requests and limits (chart 0.7.0, "Resource defaults" in the [chart README](https://github.com/plan4better/goat/blob/main/deploy/helm/goat/README.md); memory limits as in GOAT's own production, e.g. core and web 1Gi, geoapi, processes, catalog, the print and the tools worker 4Gi, the Windmill server 2Gi). Override them in `helm_values`, and set requests and limits together: Helm merges maps, so a limit set alone keeps the chart's request (and a request above its limit fails the upgrade). `resources: null` drops a service's defaults. geoapi's DuckDB budgets (`GEOAPI_DUCKLAKE_POOL_SIZE`, `GEOAPI_DUCKDB_MEMORY_LIMIT`, set by the chart) must fit its limit; raise them together.

```yaml
inv_addons:
  goat:
    helm_values:
      windmill:
        workers:
          tools:
            resources:
              requests: { memory: 2Gi, cpu: 500m }
              limits: { memory: 8Gi }
```

**Caveat:** `helm_values` overrides only the *chart* values — never the task-managed resources derived from inventory keys. Cross-cutting values (S3 endpoint, subdomain, DB host) must NOT be set via `helm_values`: the chart would see the override but the S3 gateway Ingress, ExternalName Service, Certificate and Keycloak client would keep the inventory-derived values and silently diverge.

### Object storage

`inv_addons.goat.s3.mode` picks where GOAT keeps uploads, exports, printed reports and uploaded images:

- **`provision` (default)**: the GOAT chart's [Garage](https://garagehq.deuxfleurs.fr/) (`garage.enabled`), a single-node S3 store in the goat namespace on two volumes, `goat-garage-meta` (1Gi) and `goat-garage-data` (`s3.pvc_size`, default 10Gi). The addon generates its key pair once, in Garage's format, into the Secret `goat-s3-credentials`; the chart's Job `goat-garage-init` imports it after every install and upgrade and creates the buckets `s3.bucket` and `goat-assets`. One node, no replication: back up its data volume (see [Backup](#backup)), or attach a store that replicates.
- **`attach`**: any pre-existing S3-compatible endpoint, see [External S3](#external-s3-compatible-object-storage).

#### Supported backends

| backend | `mode` | notes |
|---|---|---|
| Garage, run by the GOAT chart (default) | `provision` | single node, not HA |
| MinIO elsewhere in cluster/network | `attach` | bucket must pre-exist |
| Dell ECS | `attach` | CA chain and CORS origin are operator-side |
| **AWS S3** | **unsupported** | virtual-hosted-only buckets break the `/<bucket>` gateway; separate feature |

**Browser-facing endpoint = public GOAT URL, always.** There is no `public_endpoint` variable to set. In both modes the addon deploys a `/<bucket>` Ingress on the public GOAT host; presigned URLs returned by goat-core embed that URL as `S3_PUBLIC_ENDPOINT_URL`. In `provision` mode the Ingress backend is Garage's Service (`goat-garage`, S3 API port); in `attach` mode it is a namespace-local `ExternalName` Service (`goat-s3-upstream`) that resolves to `endpoint`. This keeps the browser origin equal to the GOAT origin — no second host to cover with a TLS cert, no cross-origin CORS surface.

#### Uploaded images (`/goat-assets`)

Images uploaded into GOAT (organization and profile pictures, images in projects and dashboards) are public reads. In provision mode core stores them in Garage's bucket `goat-assets`, which Garage serves as a website, and links them as `<public GOAT URL>/goat-assets/<key>`. Garage's website endpoint picks the bucket by the `Host` header, so the addon's Ingress `goat-s3-assets` strips the `/goat-assets` prefix and sends `Host: goat-assets.web.garage.internal` (an ingress-nginx regex rewrite). The addon creates it on ingress-nginx only. On another controller, route `/goat-assets/<key>` to the `goat-garage` Service, port `web`, as `/<key>` with that `Host`; without it uploaded images do not load.

Attach mode needs no second bucket and nothing public on the attached store: core keeps uploaded images in `s3.bucket` under `assets/` (`ASSETS_SERVED_BY_CORE`, GOAT v3.0.9 or later) and links them as `<public GOAT URL>/core/api/v2/asset/file/<key>`. That route answers with a redirect to a short-lived signed URL on the `/<bucket>` gateway, so the credentials need no rights beyond those listed under the preconditions below. On ingress-nginx, `goat-s3-assets` sends `/goat-assets/<key>` to the same route, so links saved in provision mode keep working after a switch to attach.

#### External S3-compatible object storage

Set `inv_addons.goat.s3.mode: attach` to turn off the chart's Garage and point every S3-consuming service (goat-core, geoapi, processes, all three goatlib-running windmill workers, DuckLake bootstrap) at a pre-existing endpoint. Supported backends are enumerated in [§Supported backends](#supported-backends).

```yaml
inv_addons:
  goat:
    s3:
      bucket: "goat"
      mode: "attach"
      endpoint: "https://object.ecs.example.com"
      region: "us-east-1"
      # Keep the two values in ansible-vault under the `secrets` tree.
      access_key: "{{ secrets.inv_addons.goat.s3_access_key }}"
      secret_key: "{{ secrets.inv_addons.goat.s3_secret_key }}"
```

**Preflight.** In attach mode the addon checks the storage before it installs anything that depends on it: `tasks/s3_attach.yml` runs a short Job (`goat-s3-preflight`, GOAT's core image) in the goat namespace that lists one key of the bucket with boto3, path-style, the configured region and, with `customCACert`, the same CA bundle the services get. A run whose endpoint is unreachable, whose TLS fails, whose bucket does not exist or whose keys are wrong or may not list the bucket stops there with the error and what to fix (e.g. `NoSuchBucket`, `InvalidAccessKeyId`, `SignatureDoesNotMatch`, `AccessDenied`). The check reads only; write and delete permissions are not tested. The Job may take up to 15 minutes, pulling the core image included. When it does not finish in that time, the run stops with a different message: that is no answer from the storage (image pull, scheduling, a transient outage), and a re-run may succeed.

Preconditions:

1. **`access_key` + `secret_key` supplied** via inventory (source them from ansible-vault under `secrets.inv_addons.goat.s3_access_key` / `s3_secret_key`). The addon materializes them into a `goat-s3-credentials` Secret in the goat namespace on every run — no manual `kubectl create secret` step and no cross-namespace secret references. If your credentials live in a Secret elsewhere, copy the values into the vault once; the addon owns the in-namespace Secret from then on.
2. **Bucket** identified by `s3.bucket` **already exists** at the endpoint. The addon creates no bucket in attach mode — attached credentials are typically scoped to a specific bucket without `s3:CreateBucket`, and the bucket is provisioned by the operator (Terraform, cloud console, `mc mb`, etc.) before the playbook runs. The credentials must have `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, and `s3:ListBucket` on this bucket (uploads, prints and thumbnails are stored there).
3. **`endpoint`** is reachable from the goat namespace (in-cluster DNS or public network). Used by boto3 / DuckDB inside pods, and by the `goat-s3-upstream` `ExternalName` Service (a plain DNS alias — `endpoint` must be a hostname, not an IP literal). Operator-side firewall rules must permit egress from the goat namespace to the storage host; the preflight fails the run when they do not.
4. **`region`** matches how SigV4 is signed on the target. Both MinIO and Dell ECS verify it, so it must match whatever the endpoint advertises.
5. **CORS at the storage backend.** Because the browser talks to the S3 via the `/<bucket>` path on the public GOAT host, the presigned request looks (to the pod that ultimately handles it) like `Host: <public GOAT host>` — this is set explicitly through `nginx.ingress.kubernetes.io/upstream-vhost` so SigV4 validates. If the attached S3 enforces a CORS allow-list on the request `Origin`, the operator must add the public GOAT origin (`https://<subdomain>.<DOMAIN>`) to it. Symptom when missing: uploads fail with an opaque CORS error against a healthy backend. Out of scope for the addon.

**Layer data is not in S3.** The DuckLake catalog lives in Postgres (`ducklake.*`), and its parquet files — every user layer — live on the shared data volume `goat-ducklake-data` (`/app/data/ducklake`), not in the bucket. Back it up (see [Backup](#backup)) and size it for the layer data; `inv_addons.goat.ducklake.pvc_size` defaults to 20Gi.

#### Switching from provision to attach

The addon does not move data. Attach mode turns Garage off: Helm removes its Deployment and Service but keeps its volumes (`helm.sh/resource-policy: keep`), and `goat-s3-credentials` gets the attached store's keys. So copy the bucket while Garage still runs, in a quiet moment, since uploads made after the copy stay behind:

1. Copy the bucket to the new store, from a pod in the goat namespace, with Garage's keys from `goat-s3-credentials`:

   ```sh
   NS=<env>-goat-stack
   key() { kubectl -n "$NS" get secret goat-s3-credentials -o jsonpath="{.data.$1}" | base64 -d; }
   kubectl -n "$NS" run s3copy --rm -it --restart=Never --image=docker.io/rclone/rclone:1.71 \
     --env=RCLONE_CONFIG_OLD_TYPE=s3 --env=RCLONE_CONFIG_OLD_PROVIDER=Other \
     --env=RCLONE_CONFIG_OLD_ENDPOINT=http://goat-garage:3900 --env=RCLONE_CONFIG_OLD_REGION=garage \
     --env=RCLONE_CONFIG_OLD_FORCE_PATH_STYLE=true \
     --env=RCLONE_CONFIG_OLD_ACCESS_KEY_ID="$(key access_key)" \
     --env=RCLONE_CONFIG_OLD_SECRET_ACCESS_KEY="$(key secret_key)" \
     --env=RCLONE_CONFIG_NEW_TYPE=s3 --env=RCLONE_CONFIG_NEW_PROVIDER=Other \
     --env=RCLONE_CONFIG_NEW_ENDPOINT=https://object.ecs.example.com --env=RCLONE_CONFIG_NEW_REGION=us-east-1 \
     --env=RCLONE_CONFIG_NEW_FORCE_PATH_STYLE=true \
     --env=RCLONE_CONFIG_NEW_ACCESS_KEY_ID=<new-access-key> \
     --env=RCLONE_CONFIG_NEW_SECRET_ACCESS_KEY=<new-secret-key> \
     --command -- sh -c 'rclone copy old:goat new:goat && rclone copy old:goat-assets new:goat/assets \
       && rclone check old:goat new:goat && rclone check old:goat-assets new:goat/assets'
   ```

   (`goat` is `s3.bucket` on both sides.) The uploaded images go under `assets/` in the new bucket, where attach mode reads them; links saved before the switch keep working through `goat-s3-assets` (ingress-nginx).
2. Set `s3.mode: attach` (with endpoint and keys) and run the addon.
3. Once GOAT works against the new store, delete Garage's volumes. This deletes the copy in Garage:

   ```sh
   kubectl -n <env>-goat-stack delete pvc/goat-garage-meta pvc/goat-garage-data
   ```

The `goat-s3` and `goat-s3-assets` Ingresses exist in both modes and are updated in place.

#### Moving from MinIO

Earlier addon versions ran their own MinIO in provision mode. MinIO no longer publishes free images, so GOAT now uses the chart's Garage, and the first run of this version moves GOAT onto it (or onto the attached store in attach mode). It deletes nothing: the MinIO keeps running on its volume `minio-data`, which holds every upload made before the move, with its keys copied to the Secret `goat-minio-legacy-credentials` (in provision mode `goat-s3-credentials` gets new keys in Garage's format). Every run reports while the MinIO is still there. To move its data:

1. Copy the bucket into Garage from inside the MinIO pod, which ships `mc`:

   ```sh
   NS=<env>-goat-stack
   key() { kubectl -n "$NS" get secret "$1" -o jsonpath="{.data.$2}" | base64 -d; }
   kubectl -n "$NS" exec deploy/minio -- sh -c "
     mc alias set old http://localhost:9000 '$(key goat-minio-legacy-credentials access_key)' '$(key goat-minio-legacy-credentials secret_key)' &&
     mc alias set garage http://goat-garage:3900 '$(key goat-s3-credentials access_key)' '$(key goat-s3-credentials secret_key)' --api S3v4 --path on &&
     mc mirror old/goat garage/goat"
   ```

   `goat` is `s3.bucket`. In attach mode, set the attached store as the target instead of `garage` (its endpoint and keys, and `new/<bucket>` in `mc mirror`). An install that switched to attach with an earlier addon version has no `goat-minio-legacy-credentials`: its MinIO still runs with its old keys until it restarts, so read them from the pod (`kubectl -n "$NS" exec deploy/minio -- printenv MINIO_ROOT_USER MINIO_ROOT_PASSWORD`).
2. Once GOAT works with the copied files, remove the MinIO, its volume and its keys. This deletes the old uploads:

   ```sh
   kubectl -n <env>-goat-stack delete deployment/minio service/minio job/goat-bucket-init --ignore-not-found
   kubectl -n <env>-goat-stack delete pvc/minio-data secret/goat-minio-legacy-credentials
   ```

### Restricting login (`keycloak.limit_access`)

The addon supports two access modes:

- **Open (default, `limit_access: false`).** Every user in the Keycloak realm can log into GOAT. Who may set up an organization, and how a stray one is removed, is under [Organizations and API tokens](#organizations-and-api-tokens).
- **Gated (`limit_access: true`).** For deployments with a single managed organization. The addon provisions a client role (`gate_role`) on the GOAT client, a group (`gate_group`) with that role mapped, and a dedicated browser flow (`flow_alias`) bound to the client. Users outside the group cannot authenticate against GOAT and see `deny_message`. Other realm clients are unaffected.

```yaml
inv_addons:
  goat:
    keycloak:
      limit_access: true
```

Notes and caveats:

1. **The group starts empty.** The addon creates the group but does not manage membership. Add users in the Keycloak admin console. Until the first member is added, nobody can log into GOAT, including the intended org owner.
2. **Custom browser flows are not carried over.** The gate flow is built from scratch (Cookie, Identity Provider Redirector, Organization, username/password form, gate). Customisations in the realm's default browser flow (OTP, WebAuthn, custom authenticators) are absent from the gate flow and must be re-added manually.
3. **Disabling leaves state behind.** Setting `limit_access: false` only unbinds the flow from the client. Role, group and flow stay in the realm. The group then looks like it controls GOAT access but does not. Delete manually if needed.
4. **The flow is created once.** Later changes to `deny_message` or `gate_role` do not propagate into an existing flow. Edit it in Keycloak or delete the flow and re-run the playbook.

**Onboarding runbook (gate on).** Order matters:

1. Invite the user in GOAT by email (as org admin, via the GOAT UI).
2. Add the user to the gate group in Keycloak.
3. The user logs in and accepts the invitation.

Invite first. A user who logs in without a pending invitation is prompted to create an own organization, which cannot be undone without deleting the user (see open mode above).

**Offboarding runbook.** Removing a user from the gate group does not end their GOAT sessions: the gate runs at authentication time only, and a signed-in browser keeps refreshing its tokens without passing it again. Remove the user from the group, then end their sessions in Keycloak (Users, the user, Sessions, Sign out). Access tokens already issued stay valid until they expire (the realm's access token lifespan). GOAT's APIs accept only tokens issued to `keycloak.allowed_clients`, so a token the user obtains through another client of the realm does not work on them. To cut a user off from every client of the realm at once, disable the user in Keycloak.

### Organizations and API tokens

Both settings need GOAT v3.0.9 or later.

**Organizations (`keycloak.organization_creation`).** A GOAT user belongs to exactly one organization. While this is `true` (default), someone who logs in without an invitation can set up an organization of their own, which then keeps them from joining yours. So: let the first admin log in and create your organization, then set it to `false` and run the addon again. From then on users without an invitation see a page that waits for one, and only accounts with the realm role `superuser` can create organizations.

- **Onboarding:** invite the user in GOAT (with the login gate on, also add them to `gate_group`); they log in and accept the invitation.
- **Offboarding:** remove them from `gate_group` and end their sessions in Keycloak. A user stays signed in until their sessions end.
- **A stray organization** that holds nothing but its one user is removed by an account with the realm role `superuser` through core's `DELETE /api/v2/organizations/{id}/permanent`. The user keeps their account and can then be invited. Anything in it (layers, projects, other members, pending invitations) makes the call refuse and name what is left.

**API tokens (`keycloak.allowed_clients`).** The Keycloak clients whose tokens core, geoapi, processes and catalog accept, matched against the token's `azp` (the client Keycloak issued it to). Default: GOAT's own client (`client_id`), so a token another client of the shared realm hands out is refused, also for a user the login gate keeps out. The list must include `client_id` (the run refuses otherwise: core would not start); `[]` accepts every client of the realm.

## Authentication

`global.auth` is on: core, web, geoapi, processes and catalog all validate Keycloak tokens from the civitas realm, read from the `goat-keycloak-creds` Secret that `tasks/keycloak.yml` writes. core also uses the `goat-web` client's **service account** to read and update users (profile enrichment, profile edits, account deletion); the addon enables it and maps the realm-management roles `view-users` and `manage-users` onto it on every run.

Check after an install: `kubectl -n <ns> exec deploy/goat-core -- printenv AUTH` prints `true`, and an API call without a token (e.g. `curl -i https://<host>/core/api/v2/users/profile`) answers `401`.

GOAT accepts a token only if it is valid for the realm and was issued to one of `keycloak.allowed_clients` (see above). Removing someone's access still happens in Keycloak, with or without the login gate: end their sessions and, to block them at once, disable the user (see the offboarding runbook above).

## Routing base data (street network, public transport)

Heatmaps, catchments and the PT tools read their base data from the shared volume. GOAT v3.0.3+ ships the Windmill task `f/goat/tasks/sync_base_data` for it. Its weekly schedule is created switched off, so run it once after the install, and again for updates (a run that finds every data set current downloads nothing). The task verifies every file (sha256), switches versions in one step and keeps the previous version.

**Source.** By default the task downloads from GOAT's public base data server, `https://goat-base-data.plan4better.de/`. `inv_addons.goat.base_data.source_url` sets another default source, e.g. an internal mirror: the addon passes it to the tools worker as `GOAT_BASE_DATA_URL`, which the task reads from GOAT v3.0.5 on, for manual and scheduled runs alike. On older releases pass the source in the arguments instead (`sources`, below).

**Run it** from the Windmill UI (with [`windmill.ingress`](#windmill-ui-windmillingress) on: `f/goat/tasks/sync_base_data`, Run), or through Windmill's API with the token the chart keeps in the `goat-windmill-token` Secret:

```sh
NS=<env>-goat-stack
TOKEN=$(kubectl -n "$NS" get secret goat-windmill-token -o jsonpath='{.data.token}' | base64 -d)
kubectl -n "$NS" port-forward svc/goat-windmill-server 8000:80 &
# Arguments: limit the download to your region (H3 cells around the bbox),
# and optionally name the source, e.g. {"sources": [{"url": "https://mirror.example.org/goat-base-data/"}]}
curl -fsS -X POST "http://127.0.0.1:8000/api/w/goat/jobs/run/p/f/goat/tasks/sync_base_data" \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"bbox": [<min_lon>, <min_lat>, <max_lon>, <max_lat>]}'
# prints the job id; follow the run in the tools worker's log:
kubectl -n "$NS" logs deploy/goat-windmill-worker-tools -f
```

Without `bbox` the task fetches the full data set (tens of GB). Other arguments: `datasets` (default `["street_network", "public_transport"]`), `ring`, `pin`; see the task's description in Windmill.

**Air-gapped:** build a copy on any machine with internet access, upload it to your bucket (or any HTTPS server the cluster reaches), and point the task at it. For an `s3://` source the worker's own `S3_*` settings (and `AWS_CA_BUNDLE`) are used:

```sh
docker run --rm --entrypoint /venv/bin/python -v "$PWD:/out" \
  ghcr.io/plan4better/goat/windmill-worker-tools:v3.0.9 \
  -m goatlib.tasks.sync_base_data mirror --to /out/goat-base-data --bbox=<min_lon>,<min_lat>,<max_lon>,<max_lat>
# upload ./goat-base-data to the bucket, e.g. under goat/goat-base-data/, then set
#   base_data.source_url: "s3://goat/goat-base-data/"   (GOAT v3.0.5+)
# or run the task with {"sources": [{"url": "s3://goat/goat-base-data/"}]}
```

## Connection pooler (`db.pooler`)

Off by default, and GOAT does not need one: without a pooler it holds a few dozen connections at rest. With it on, GOAT's own queries go through a PgBouncer in transaction mode: core, geoapi, processes and the jobs of the print, tools and workflows workers (`postgresql.external.host`, `POSTGRES_SERVER`). Their queries then share the pooler's server connections instead of every pod holding a pool of its own.

Two kinds of connections stay on the database itself:

- **DuckLake's catalog** (`postgresql.external.ducklakeHost`, `DUCKLAKE_POSTGRES_SERVER`: geoapi, processes, their DuckLake init and the workers). A DuckLake attach keeps its session open for as long as the service runs, which a transaction pooler breaks.
- **Windmill** (`db.windmill_host`): its server and workers keep small pools of their own (the server 3 connections, every worker 3), which a pooler would not shrink.

**GOAT behind a stock pooler.** The addon sets the chart's `postgresql.external.transactionPooler` (`POSTGRES_TRANSACTION_POOLER`) with the pooler. GOAT then prepares no statements on the server and sends no TCP keepalive settings at connect, so a PgBouncer with stock settings works: civitas-core's pooler image (`registry.opensource.zalan.do/acid/pgbouncer:master-32`, PgBouncer 1.22.0, `max_prepared_statements` 0, `ignore_startup_parameters = extra_float_digits,options`) needs no changes. The setting exists from GOAT v3.0.9 on; the run refuses the pooler with an older `release`.

```yaml
inv_addons:
  goat:
    db:
      pooler:
        enable: true
```

The addon then sets `enableConnectionPooler: true` on the central-db resource (the shared resource it already patches with GOAT's databases), waits for the operator's `central-db-pooler` Deployment, and points GOAT at `central-db-pooler.<db-namespace>.svc.cluster.local`.

**A PgBouncer of your own** in transaction mode: set `db.pooler.host` to it; the addon then leaves central-db alone. With a `db.host` other than central-db, `db.pooler.host` is required.

Limits:

- The pooler must listen on the database's port (`db.port`): DuckLake reaches the database on the same port. Zalando's pooler does (5432).
- `db.geoapi_host` / `db.geoapi_port`, when set, still win for geoapi.
- Switching `enable` off points GOAT at the database again but leaves the pooler running: other components may use it by then. Remove `enableConnectionPooler` from the central-db resource yourself when nothing does.

**Check on the cluster after enabling** (not yet run on a civitas cluster):

```sh
NS=<env>-goat-stack
# 1. GOAT is up, points at the pooler and knows it
kubectl -n "$NS" rollout status deploy/goat-core deploy/goat-geoapi deploy/goat-processes
kubectl -n "$NS" exec deploy/goat-core -- printenv POSTGRES_SERVER POSTGRES_TRANSACTION_POOLER
# 2. load through it: sign in to GOAT, copy the access token from a request's
#    Authorization header (browser developer tools), then
for i in $(seq 100); do curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer $TOKEN" https://<goat host>/core/api/v2/users/profile & done | sort | uniq -c
#    expect only 200; open a project with a few layers (geoapi) and run a tool, e.g. Buffer (processes, tools worker)
# 3. no pooler errors in the logs
for d in goat-core goat-geoapi goat-processes goat-windmill-worker-tools; do
  kubectl -n "$NS" logs deploy/$d --since=15m | grep -iE 'prepared statement|unsupported startup parameter' ; done
```

## Windmill UI (`windmill.ingress`)

Off by default: `processes` reaches Windmill in-cluster, and nobody needs its UI to use GOAT. For operators who want to watch jobs, run tasks (e.g. [`sync_base_data`](#routing-base-data-street-network-public-transport)) or edit schedules:

```yaml
inv_addons:
  goat:
    windmill:
      ingress:
        enable: true
        # subdomain: "goat-windmill"     # default: <web.subdomain>-windmill
```

Windmill cannot be served under a path prefix, so it gets its own host, `<subdomain>.<INGRESS_DOMAIN | DOMAIN>` (needs a DNS record like the GOAT host), with its own cert-manager `Certificate` (`<host>-tls`, from `tasks/certificate.yml`) and the same ingress class, `ssl-redirect` and `whitelist-source-range` (`user_whitelist`) as GOAT's ingresses. Windmill's `BASE_URL` becomes `https://<subdomain>.<DOMAIN>` (`windmill.ingress.public_url` overrides it), which its links and SSO callbacks are built from. Switching it off removes the Ingress and the Certificate; the TLS Secret stays.

**Login.** Windmill keeps its own accounts; a login through the realm (OIDC) is not set up, since only operators use this UI. The chart's bootstrap set the superadmin `admin@windmill.dev` to a generated password: `kubectl -n <ns> get secret goat-windmill-admin-cred -o jsonpath='{.data.password}' | base64 -d`. Whoever logs in as an admin of the `goat` workspace can run any script with the workers' environment, which includes GOAT's database password and S3 keys: keep the accounts few, and narrow the source range for this host alone if you can:

```yaml
inv_addons:
  goat:
    helm_values:
      windmill:
        server:
          ingress:
            annotations:
              nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8"
```

**Keycloak login (SSO).** Windmill's Community Edition supports SSO with Keycloak, limited to 10 SSO users (Windmill: "Without Enterprise Edition, the number of SSO users is limited to 10"). It is configured in Windmill's instance settings, not through its environment, so the addon does not set it up. To use it:

1. In the civitas realm, create a confidential OpenID Connect client (e.g. `goat-windmill`) with the redirect URI `https://<windmill host>/user/login_callback/keycloak` and the client scope `offline_access`.
2. In Windmill (superadmin), Instance settings, SSO/OAuth: add Keycloak with the realm URL (`<keycloak>/realms/<realm>`), the client id and its secret.
3. In the same settings, require users to be added to Windmill manually before they can sign in through SSO; otherwise every user of the realm can sign in. The GOAT login gate (`keycloak.limit_access`) applies to the GOAT client only.

## Backup

The addon backs nothing up itself, like the other CIVITAS/CORE components and add-ons: backups are the platform's job. GOAT keeps its data in four places:

| What | Where | Back it up with |
|---|---|---|
| Projects, layer metadata, users, organizations, and the DuckLake catalog (schema `ducklake`) | the goat database on central-db | central-db's logical backup (`inv_central_db.enable_logical_backup`, `inv_op_stack.postgres_operator.logical_backup`) |
| Every user layer: the DuckLake parquet files, plus the catalog mirror and the routing base data | the shared volume `goat-ducklake-data` (`/app/data`) | the platform's volume backup (Velero with file-system backup, Ceph snapshots or similar) |
| Uploads, exports, printed reports, uploaded images | provision mode: Garage's volumes `goat-garage-meta` and `goat-garage-data`; attach mode: the attached bucket | provision mode: the platform's volume backup; attach mode: the S3 backend's own backup or replication |
| Windmill's scripts, schedules and job history | the windmill database on central-db | central-db's logical backup |

**With civitas-core's Velero on** (`inv_op_stack.velero.enable`), the addon registers the Schedule `<inv_addons.goat.namespace>-backup` in Velero's namespace, as civitas-core's own components do: nightly at 02:00 UTC, GOAT's namespace with its volumes (`goat-ducklake-data`, and Garage's `goat-garage-meta` and `goat-garage-data` in provision mode) through Velero's file-system backup, kept 30 days, in the platform's backup location. 02:00 UTC is after central-db's logical backup (00:30 UTC by default), so both rules below hold. Without Velero, back up the volume with the platform's own tools.

Two rules keep a backup restorable:

1. **The database before the volume.** DuckLake writes each parquet file once and never changes it; the catalog in the database says which files make up a layer. A restore needs every file the restored catalog refers to, so back up the goat database first and the volume afterwards. Files on the volume that the catalog does not know are ignored.
2. **Not between 00:00 and 01:00 UTC, and both on the same day.** GOAT's DuckLake maintenance (00:00) deletes the files of expired snapshots, and its compaction (00:30) rewrites small files into larger ones.

**Restore.** Stop what writes DuckLake, then restore the database and the volume from the same night:

```sh
NS=<env>-goat-stack
kubectl -n "$NS" scale deploy goat-geoapi goat-processes goat-windmill-worker-tools \
  goat-windmill-worker-workflows goat-windmill-worker-print --replicas=0
# restore the goat database (or only its `ducklake` schema) and the volume
# goat-ducklake-data with the platform's tools, then scale back to 1
```

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
| **current** | v1.7.x+ | v0.7.0 | v3.0.9 |
| v0.3.0      | v1.7.x+ | v0.4.x | v2.4.60 |
| v0.2.0      | v1.5.x+ | v0.3.x | mixed (`latest` / `f59d1e3` / `v2.4.36`) |
| v0.1.x      | v1.5.x+ | v0.1.x |  |

Addon version is independent of civitas-core's — tag the addon on its own semver.

### Upgrading to chart 0.7.0

This addon version installs chart 0.7.0 or later only: `tasks/preflight.yml` stops the run before any change when `chart.version` (or a local chart's own version) is older. Upgrade the addon and the chart together.

1. **Redis** is the chart's own Deployment on the official `redis` image instead of the Bitnami sub-chart. The password stays (same Secret `goat-redis`), geoapi's cache starts empty once. The Bitnami replicas' volumes are not deleted by Helm; remove them after the upgrade: `kubectl -n <ns> delete pvc -l app.kubernetes.io/name=redis,app.kubernetes.io/instance=goat,app.kubernetes.io/component=replica`. Sub-chart keys in `helm_values` (`redis.architecture`, `redis.master.*`, `redis.replica.*`, `global.redis.*`) are no longer read.
2. **Memory limits.** Every service now runs with the chart's requests and limits (see [Inventory key vs. `helm_values`](#inventory-key-vs-helm_values)). Raise them in `helm_values` before upgrading if a service used more.
3. **geoapi and processes run as root** (chart default `podSecurityContext.runAsUser: 0`), like the Windmill workers, so their DuckLake init can create the catalog directory on a fresh volume. The addon sets no `podSecurityContext` of its own. See the pod security caveat below.
4. **Worker environment.** The chart now gives the print, tools and workflows workers the GOAT database, DuckLake and the Keycloak client, and builds `WHITELIST_ENVS` itself; the addon no longer sets a fixed list. A `WHITELIST_ENVS` in `helm_values` would replace the built list; remove it.
5. **Windmill 1.751** (GOAT v3.0.9 on). Its server migrates the Windmill database on its first start, one way: back up the `windmill` database first, and don't go back to older images afterwards.
6. **A failed upgrade is not retried by running the addon again.** civitas-core's Helm step compares the chart version and the values with the release and does nothing when they match, also when that release's last upgrade failed. Roll back to the last working revision first (`helm -n <ns> history goat`, then `helm -n <ns> rollback goat <revision>`), then run the addon again.

## Image references (mirror-friendly)

Every container image this addon spins up — the chart-deployed services, Garage included, and the addon's own Jobs (the S3 preflight runs the core image) — is enumerated in `vars/software_references.yml` under `software.addon_goat.images.*`. Override `registry:` per image in your inventory to point at a private mirror.

```sh
yq '.software.addon_goat.images[] | "\(.registry)/\(.repository):\(.tag)"' \
   vars/software_references.yml
```

**Caveat.** Civitas-core's own `tools/extract-images` and `tools/harbor` playbooks currently only enumerate images under a top-level `software:` key, whereas this addon uses `software.addon_goat:`. As a result the addon's images are not picked up by that tooling today. Track this as a known integration gap.

## Known caveats

- **`subdomain` and `web.public_url` are load-bearing across the whole stack**: `inv_addons.goat.web.subdomain` composes the ingress-visible hostname (`<subdomain>.<INGRESS_DOMAIN>`) and the browser-facing URL (`https://<subdomain>.<DOMAIN>`); `web.public_url` optionally overrides the latter. Together they drive every `NEXT_PUBLIC_*` env baked into the goat-web bundle, `CLIENT_URL` on goat-core, the Keycloak `redirectUris` / `webOrigins` / `postLogoutRedirectUris`, and `S3_PUBLIC_ENDPOINT_URL`. The ingress name must resolve to the ingress load balancer — DNS record and cluster-issuer both have to agree. Changing subdomain or domain rotates the addon into a new host-derived TLS secret (`<ingress-host>-tls`); the OLD secret lingers in the namespace after the change and should be deleted manually to reclaim quota. The Keycloak client is PUT on every run against the same URL template, so hostname changes now propagate to `redirectUris` immediately — no more `invalid parameter: redirect_uri` after a rename.

- **The `cacert` mount and CA env vars are conditional on `inv_k8s.ingress.ca_path`**: the `REQUESTS_CA_BUNDLE` / `SSL_CERT_FILE` (geoapi, processes) env vars, together with the `cacert` `extraVolumes` / `extraVolumeMounts` on those services (core and catalog too — with auth on, every Python service fetches Keycloak's keys over HTTPS), are only rendered when `inv_k8s.ingress.ca_path` is set — the same condition that gates the `cacert` ConfigMap in `tasks/namespace.yml`. With the ConfigMap present the addon also sets the chart's `global.caBundle` to it, which mounts it at `/etc/goat/ca/ca.pem` in core, web (`NODE_EXTRA_CA_CERTS`) and the print, tools and workflows workers (`GOAT_CA_BUNDLE`). On clusters that use an ACME-issued certificate (chain trusted by the Node / OpenSSL default store), leave `ca_path` unset and no bundle is mounted. Previously the env vars were always emitted and goat-web logged `Warning: Ignoring extra certs from /etc/ssl/cacert/cacert.crt … No such file or directory` at every start. Do NOT substitute `kube-root-ca.crt`: that ConfigMap signs `kubernetes.default.svc`, not the public ingress cert, and is useless as a trust anchor for outbound HTTPS.

- **`inv_addons.goat.s3.customCACert` adds an external S3 CA to the same `cacert` ConfigMap**: attach-mode S3 endpoints served by a private CA cause `CERTIFICATE_VERIFY_FAILED` in every pod-side boto3 and DuckDB httpfs client. Set `inv_addons.goat.s3.customCACert` to a PEM path on the Ansible controller. The content is concatenated with `inv_k8s.ingress.ca_path` (if set) into the `cacert` ConfigMap. goat-core, geoapi and processes mount it at `/etc/ssl/cacert` and receive `AWS_CA_BUNDLE` and `CURL_CA_BUNDLE` pointing at `/etc/ssl/cacert/cacert.crt`. The windmill `print`, `tools` and `workflows` workers get it through the chart's `global.caBundle` at `/etc/goat/ca/ca.pem`, and their `AWS_CA_BUNDLE` / `CURL_CA_BUNDLE` point there (the chart renders no `extraVolumes` for workers). The chart adds both names to the workers' `WHITELIST_ENVS`, so the jobs inherit them. The S3 preflight uses the same bundle.

- **`goat-web` server-side fetches hairpin through the ingress**: the Next.js server calls `NEXT_PUBLIC_API_URL` from inside the pod (SSR of the org-creation flow), which routes back out through the same ingress that serves the browser. Until a valid cert is served on the ingress host, Node rejects the chain with `SELF_SIGNED_CERT_IN_CHAIN` and the UI silently loops on the organization-creation screen — no error in the browser console, no HTTP error visible to the user, just an endless refresh (the POST to `/organizations` on the second attempt then answers `{"detail":"User has already an organization"}` even though the frontend never got the first response). Confirm the cert served on the ingress host is trusted by the goat-web pod (`NODE_EXTRA_CA_CERTS` points at the civitas CA bundle from `inv_k8s.ingress.ca_path` — set that in inventory when the ingress cert chains up through a private CA; a real ACME cert needs no extra bundle).

- **Postgres-operator stale-password**: if civitas's Zalando postgres-operator has lost its in-memory state (K8s secret rotated but DB password unchanged), `db.yml` will hang waiting for the goat secret to appear. Restart the operator pod (`kubectl -n <operator-ns> delete pod -l app.kubernetes.io/name=postgres-operator`) and retry.

- **`storage_class` is a dict, not a string**: civitas-core's `inv_k8s.storage_class` is `{loc, rwo, rwx}`. The chart wants a single string; the rendered values file picks `loc` by default. Override `inv_k8s.storage_class.loc` (or edit `templates/goat_values.yml` in a fork) for multi-node setups that need `rwx` storage.

- **`local-path` for the DuckLake PVC is single-node ONLY**: `local-path` is not a CSI driver and does not reject `ReadWriteMany` at bind time — it silently binds RWX on any node, and each node then gets a *separate empty directory*. On multi-node clusters this means `layer_import` (worker-tools) writes parquet files that `geoapi` cannot see, with no error anywhere: the tile requests just return empty results. Use a real RWX class (CephFS / NFS / EFS) on multi-node, or pin the data-touching pods to a single node. The addon accepts `inv_addons.goat.ducklake.storage_class` as an override so operators can point at a filesystem-backed class without moving `inv_k8s.storage_class.rwx` globally — block-backed classes (Ceph RBD, EBS, GCE PD, Azure Disk) cannot satisfy RWX and the PVC will sit Pending indefinitely with only a generic "waiting for external provisioner" event.

- **Pod security: GOAT runs root containers.** geoapi and processes (chart 0.7.0 default `runAsUser: 0`), and the Windmill workers run as root: they share a volume a fresh RWX provisioner hands out owned by root. Garage's image runs as root as well. With `inv_k8s.podSecurityAdmission.enable`, civitas-core's namespace helper labels the goat namespace `pod-security.kubernetes.io/enforce: restricted`, which rejects them. Leave it off for the goat namespace, or create the namespace yourself (`ns_create: false`) with a `baseline` or `privileged` level. To run geoapi and processes as another user, set `<service>.podSecurityContext.runAsUser` in `helm_values` and make sure that user may write the whole data volume.

- **`goat-core` has no curl**: the image is stripped to essentials. Don't write tasks that `kubernetes.core.k8s_exec` shell-out into the container; rely on Kubernetes probes instead.

- **Ansible Jinja2 in dict keys**: if you ever need to add a third `preparedDatabases` entry with a name derived from a variable, note that Ansible does NOT evaluate Jinja inside YAML dict keys. Workaround in `db.yml`: a `set_fact` + a single Jinja dict-literal expression.

## License

[EUPL-1.2](LICENSE) — same as civitas-core.
