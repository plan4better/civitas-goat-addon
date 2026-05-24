# CIVITAS-GOAT addon

Civitas-core addon that installs the GOAT Helm chart and wires it to civitas's
own Postgres (central-db) and Keycloak.

## Phase 1 status

This addon installs the GOAT Helm chart (`oci://ghcr.io/plan4better/charts/goat`) into
civitas with auth disabled and only `core` + `web` deployed. The remaining services
(`geoapi`, `accounts`, `processes`) are opt-in in the chart and remain off by default
for civitas until their bootstrap requirements (DuckLake init for geoapi/processes,
private GHCR pull secret for accounts) are addressed.

**What works**: namespace creation, postgres user/db provisioning via civitas central-db,
helm install, `/api/healthz` smoke check.

**What's deferred to Phase 2**: Keycloak client registration (`02_keycloak_client.yml`),
APISIX route registration (`04_apisix_routes.yml`).

## What this addon does

1. Creates a `<env>-goat-stack` Kubernetes namespace.
2. Adds a `goat` role and database to civitas's central-db (Zalando postgres-operator).
3. Renders a chart values file pointing GOAT at civitas's central-db.
4. Installs the GOAT Helm chart from `oci://ghcr.io/plan4better/charts/goat`.
5. Verifies `core /api/healthz` returns 200.

**Phase 1 limitation:** only GOAT `core` is deployed. Auth is disabled
(`core.auth.enabled: false`). Keycloak client registration and APISIX route
registration arrive in Phase 2.

## How to use

In your civitas-core fork:

```sh
git submodule add https://github.com/plan4better/civitas-goat-addon.git \
                 core_platform/addons/goat_addon
```

In your inventory file:

```yaml
inv_addons:
  import: true
  addons:
    - "addons/goat_addon/tasks.yml"
  goat:
    enable: true
    chart:
      ref: "oci://ghcr.io/plan4better/charts/goat"
      version: "0.1.0"
```

Then run the normal civitas playbook with the `addons` tag:

```sh
ansible-playbook -i cc_cli_inventory.yml core_platform/playbook.yml --tags addons
```

## Compatibility

- CIVITAS/CORE: v1.5.x or newer
- GOAT Helm chart: v0.1.x

## Known caveats

- **Postgres-operator stale-password**: if civitas's Zalando postgres-operator has lost
  in-memory state (k8s secret rotated but DB password unchanged), the `01_db.yml` task
  will hang waiting for the goat secret. Restart the operator:
  `kubectl -n cc-loc-operation-stack delete pod -l app.kubernetes.io/name=postgres-operator`
  then retry.
- **Ansible Jinja2 in dict keys**: `01_db.yml` uses a `set_fact` dict-literal pattern
  because Ansible does not evaluate Jinja in YAML dict keys. Don't refactor naively.
- **Pre-existing `postgrescluster.yml` bug**: if you run the playbook without
  `--tags addons,goat_addon`, the upstream `tasks/operation/postgrescluster.yml`
  task fails on an undefined `postgres_password` variable. Scope tags to the addon.

## License

EUPL-1.2 (same as civitas-core).
