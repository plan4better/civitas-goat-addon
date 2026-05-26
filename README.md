# civitas-goat-addon

[CIVITAS/CORE](https://docs.core.civitasconnect.digital/) Ansible addon that installs the [GOAT](https://github.com/plan4better/goat) Helm chart and wires it to civitas's own Postgres (central-db).

## Phase 2 status

This addon installs the GOAT Helm chart (`oci://ghcr.io/plan4better/charts/goat`) into civitas with **real auth** through civitas's Keycloak realm and **APISIX routing** for the API.

**What works in v0.2.x:**
- `<env>-goat-stack` namespace creation
- `goat` user/database provisioning in civitas's central-db
- Keycloak `goat-web` OIDC client provisioning in the civitas realm (idempotent)
- K8s Secret `goat-keycloak-creds` populated with `server-url`, `realm`, `client-id`, `client-secret`, `nextauth-secret`
- Helm install of GOAT chart v0.3.0 with `core.auth.enabled: true` + `web.auth.enabled: true`
- APISIX routes `/goat/api/*` (OIDC-enforced) and `/goat/web/*` (passthrough — web handles its own login)
- Readiness gate on `goat-core`

## Inventory

Full schema for the operator's `cc_cli_inventory.yml` under `inv_addons.goat`:

```yaml
inv_addons:
  goat:
    enable: true
    namespace: "{{ ENVIRONMENT }}-goat-stack"
    chart:
      ref: "oci://ghcr.io/plan4better/charts/goat"
      version: "0.3.2"
    db:
      # Optional — Postgres database names. Defaults shown.
      # Override only if you need to co-tenant multiple goat installs in
      # one Postgres cluster. NB windmill's *role* names (windmill_admin,
      # windmill_user) are hardcoded by its migrations regardless, so the
      # decoupling is partial.
      goat_db_name: "goat"
      windmill_db_name: "windmill"
    keycloak:
      # The OIDC client ID to create in the civitas Keycloak realm.
      # The realm itself is read from inv_access.tenant.realm_name.
      client_id: "goat-web"
    web:
      # Public-facing URL of the goat-web frontend. Used to set Keycloak
      # redirect URIs and the NextAuth NEXTAUTH_URL env var.
      public_url: "https://goat.{{ DOMAIN }}"
```

## How to use

In your civitas-core fork:

```sh
# Civitas-core's .gitignore excludes `core_platform/addons/`, so the addon
# is cloned directly (not added as a tracked submodule). The Ansible
# playbook reads files from the path regardless of git state.
git clone --branch v0.2.0 https://github.com/plan4better/civitas-goat-addon.git \
          core_platform/addons/goat_addon
```

In your inventory file (`cc_cli_inventory.yml` or equivalent), set the keys under `inv_addons.goat` documented in the [Inventory](#inventory) section above. Don't forget to also register the addon's `tasks.yml`:

```yaml
inv_addons:
  import: true
  addons:
    - "addons/goat_addon/tasks.yml"
  goat:
    # ...see Inventory schema above...
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

| addon | civitas-core | GOAT chart | notes |
|---|---|---|---|
| **v0.2.0** *(current)* | v1.5.x+ | v0.3.x | Adds Keycloak client + APISIX routes |
| v0.1.2 | v1.5.x+ | v0.1.x | |
| v0.1.1 | v1.5.x+ | v0.1.x | |
| v0.1.0 | v1.5.x+ | v0.1.0 | |

Addon version is independent of civitas-core's. Tag the addon based on its own semver; consult this table for compatibility.

## Known caveats

- **Postgres-operator stale-password**: if civitas's Zalando postgres-operator has lost its in-memory state (k8s secret rotated but DB password unchanged), `01_db.yml` will hang waiting for the goat secret to appear. Restart the operator and retry:
  ```sh
  kubectl -n cc-loc-operation-stack delete pod -l app.kubernetes.io/name=postgres-operator
  ```

- **Ansible Jinja2 in dict keys**: if you ever need to add a third
  `preparedDatabases` entry with a name derived from a variable, you'll find
  Ansible does NOT evaluate Jinja inside YAML dict keys. The workaround is a
  `set_fact` + a single Jinja dict-literal expression
  (`{{ {varname: {...}} }}`). `01_db.yml` used this for the `goat` key before
  both keys were hardcoded to literal `goat` + `windmill`.

- **`storage_class` is a dict, not a string**: civitas-core's `inv_k8s.storage_class` is `{loc, rwo, rwx}`. The chart wants a single string; the rendered values file picks `loc` by default. Override `inv_k8s.storage_class.loc` (or modify `templates/goat-values.yaml.j2` in a fork) for multi-node setups that need `rwx` storage.

- **Pre-existing `postgrescluster.yml` bug** in civitas-core: running the playbook without scoped tags can fail with an undefined `postgres_password` variable. Always scope to `--tags "addons,goat_addon"`.

- **`goat-core` has no curl**: the image is stripped to essentials. Don't write tasks that `kubernetes.core.k8s_exec` shell-out into the container; rely on Kubernetes probes instead.

## License

[EUPL-1.2](LICENSE) — same as civitas-core.
