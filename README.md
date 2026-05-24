# civitas-goat-addon

[CIVITAS/CORE](https://docs.core.civitasconnect.digital/) Ansible addon that installs the [GOAT](https://github.com/plan4better/goat) Helm chart and wires it to civitas's own Postgres (central-db).

## Phase 1 status

This addon installs the GOAT Helm chart (`oci://ghcr.io/plan4better/charts/goat`) into civitas. The chart's `core` and `web` services are deployed by default; the optional `geoapi`, `accounts`, and `processes` services remain off until their bootstrap requirements (DuckLake init for `geoapi`/`processes`, private GHCR pull secret for `accounts`) are met — see the chart's README for how to enable them.

**What works in v0.1.x:**
- `<env>-goat-stack` namespace creation
- `goat` user/database provisioning in civitas's central-db (Zalando postgres-operator's `preparedDatabases`)
- Helm install of the GOAT chart with civitas-profile values
- Readiness gate on `goat-core` (chart's readinessProbe on `/api/healthz`)

**Deferred to Phase 2:**
- Keycloak client registration (`02_keycloak_client.yml`)
- APISIX route registration (`04_apisix_routes.yml`)

Auth is disabled (`core.auth.enabled: false`) until Phase 2.

## How to use

In your civitas-core fork:

```sh
# Civitas-core's .gitignore excludes `core_platform/addons/`, so the addon
# is cloned directly (not added as a tracked submodule). The Ansible
# playbook reads files from the path regardless of git state.
git clone --branch v0.1.2 https://github.com/plan4better/civitas-goat-addon.git \
          core_platform/addons/goat_addon
```

In your inventory file (`cc_cli_inventory.yml` or equivalent):

```yaml
inv_addons:
  import: true
  addons:
    - "addons/goat_addon/tasks.yml"
  goat:
    enable: true
    namespace: "{{ ENVIRONMENT }}-goat-stack"
    chart:
      ref: "oci://ghcr.io/plan4better/charts/goat"
      version: "0.1.0"
    db:
      reuse_central: true
      db_name: goat
```

Then run the civitas playbook scoped to addons:

```sh
ansible-playbook -i cc_cli_inventory.yml core_platform/playbook.yml \
  --tags "addons,goat_addon"
```

Expected output ends with:

```
TASK [Report success] ************************
ok: [localhost] => {
    "msg": "✓ goat-core is Available in namespace <env>-goat-stack ..."
}
```

## Compatibility

| addon | civitas-core | GOAT chart |
|---|---|---|
| **v0.1.2** *(current)* | v1.5.x+ | v0.1.x |
| v0.1.1 | v1.5.x+ | v0.1.x |
| v0.1.0 | v1.5.x+ | v0.1.0 |

Addon version is independent of civitas-core's. Tag the addon based on its own semver; consult this table for compatibility.

## Known caveats

- **Postgres-operator stale-password**: if civitas's Zalando postgres-operator has lost its in-memory state (k8s secret rotated but DB password unchanged), `01_db.yml` will hang waiting for the goat secret to appear. Restart the operator and retry:
  ```sh
  kubectl -n cc-loc-operation-stack delete pod -l app.kubernetes.io/name=postgres-operator
  ```

- **Ansible Jinja2 in dict keys**: `01_db.yml` uses a `set_fact` + Jinja dict-literal pattern because Ansible does not evaluate Jinja inside YAML dict keys. Don't naively refactor to the more obvious form — it silently produces a literal `{{ goat_addon.db.name }}` key in the patched `postgresql.acid.zalan.do` CR.

- **`storage_class` is a dict, not a string**: civitas-core's `inv_k8s.storage_class` is `{loc, rwo, rwx}`. The chart wants a single string; the rendered values file picks `loc` by default. Override `inv_k8s.storage_class.loc` (or modify `templates/goat-values.yaml.j2` in a fork) for multi-node setups that need `rwx` storage.

- **Pre-existing `postgrescluster.yml` bug** in civitas-core: running the playbook without scoped tags can fail with an undefined `postgres_password` variable. Always scope to `--tags "addons,goat_addon"`.

- **`goat-core` has no curl**: the image is stripped to essentials. Don't write tasks that `kubernetes.core.k8s_exec` shell-out into the container; rely on Kubernetes probes instead.

## Phase 2 roadmap

Coming in v0.2.x:
- `02_keycloak_client.yml` — register a `goat` OIDC client in the civitas Keycloak realm using the `keycloak.openid` Ansible collection. Stores the client-secret in a K8s Secret in the goat namespace; the chart's `core.auth.enabled: true` consumes it.
- `04_apisix_routes.yml` — register `/goat/api/*` and similar routes in APISIX so the goat services are reachable through civitas's API gateway with OIDC enforcement.

## License

[EUPL-1.2](LICENSE) — same as civitas-core.
