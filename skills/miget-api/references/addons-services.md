# Add-ons and services

An **add-on** belongs to one application and dies with it. A **service** stands on
its own and can be mounted into several applications. Both are the same
underlying types — PostgreSQL, MySQL, Valkey, storage — so this file covers them
together and says which shape each endpoint takes.

Read this before creating either. Two fields the API marks optional are in
practice required (`label`, and the type's version field), and creating a
database without reading the connection-variable rules leads to writing a
`DATABASE_URL` the platform already set.

## Contents

- [Endpoints](#endpoints)
- [Creating an add-on](#create-app-addon-post-apiv1appsuuidaddons)
- [Creating a service](#create-service-post-apiv1services)
- [Walking a customer through adding a database](#walking-a-customer-through-adding-a-database)
- [Replicas and promotion](#8-create-a-postgresql-read-replica)
- [Mounting shared storage](#mount-app-to-service-post-apiv1servicesidmount_app)

## Add-on or service?

When a user asks to create a database, cache, or storage, you must first clarify *how* they want to create it. There are two primary methods:

1.  **App Addon (`POST /api/v1/apps/{uuid}/addons`)**: An addon is attached directly to a *specific, existing application*. The addon's lifecycle is tied to the app. Connection details are automatically injected as environment variables into that app.
2.  **Standalone Service (`POST /api/v1/services`)**: A service is a standalone resource that can be shared across *multiple applications* within a project. It has its own lifecycle and is not tied to any single app.

Ask which method they prefer - creating the wrong one wastes time and may require starting over.

**Example Interaction:**

> **User:** "I need a PostgreSQL database."
>
> **Agent:** "I can create a PostgreSQL database in two ways. Which do you prefer?
>
> 1.  **App Addon**: Attach it to a specific, existing app. Connection details will be injected as env vars automatically.
> 2.  **Standalone Service**: Create it as a shared resource for multiple apps in a project.
>
> If you choose 'App Addon', please provide the UUID of the application it should be attached to."

Once the user chooses, proceed to collect the required parameters for the chosen method.

For PostgreSQL databases, also ask about:
- **Creation mode**: Fresh database (default) or external replica (replicating from an existing external PostgreSQL).
- **Deployment type**: Standalone instance (default, 1 instance) or High Availability cluster (3, 5, or 7 instances with automatic failover).

---

## Services

**Services** are standalone managed services (databases, caches, storage) that can be shared across multiple apps.

- Service types: `postgres`, `shared_storage`
- Services can be mounted to apps as addons
- Services have their own lifecycle independent of any app

## Endpoints

## App Addons

- `GET /api/v1/apps/{uuid}/addons` - List app addons (includes `role` and `primary_addon_uuid` fields for PostgreSQL addons)
- `POST /api/v1/apps/{uuid}/addons` - Create addon (PostgreSQL supports `creation_mode`: `fresh` or `external_replica`, and `instances` [1, 3, 5, 7] for HA clusters)
- `GET /api/v1/apps/{uuid}/addons/{id}` - Get addon details (includes `role`, `primary_addon_uuid`, and `replicas` for PostgreSQL primaries)
- `PUT /api/v1/apps/{uuid}/addons/{id}` - Update addon (PostgreSQL supports `instances` [1, 3, 5, 7] for scaling cluster nodes, boolean `backup_enabled` and `public_access`, and a 6-field cron `scheduled_backup`)
- `DELETE /api/v1/apps/{uuid}/addons/{id}` - Delete addon (deleting a primary cascades to all its replicas; requires `apps:manage`)
- `PATCH /api/v1/apps/{uuid}/addons/{id}/state` - Change addon state (process_start/process_stop/process_restart)
- `POST /api/v1/apps/{uuid}/addons/{id}/rotate_password` - Rotate addon password (only for certain addon types - databases like PostgreSQL/MySQL, caches like Valkey)
- `POST /api/v1/apps/{uuid}/addons/{id}/create_replica` - Create a read replica of a PostgreSQL addon (primary only, optional `cpu_size`/`ram_size` params, requires `apps:operate`)
- `POST /api/v1/apps/{uuid}/addons/{id}/promote_replica` - Promote a read replica to a standalone PostgreSQL instance (replica only, requires `apps:operate`)
- `POST /api/v1/apps/{uuid}/addons/{id}/promote_external` - Promote an external replica to standalone instance or cluster by disconnecting from external source (preserves current mode, requires `apps:operate`)
- `GET /api/v1/apps/{uuid}/addons/{id}/backups` - Get addon backups (PostgreSQL primary only, not available for replicas). Answers `503` when the backup state cannot be read from the cluster: retry later, it does not mean backups are disabled. `backup_enabled` is the configured setting and changes as soon as an update is accepted; `backup_active` is what the cluster runs right now and catches up asynchronously. After enabling backups, `backup_enabled: true` with `backup_active: false` means the change is still being applied, not that it was ignored — poll until `backup_active` matches. `backup_enabled` is always `false` on the free plan
- `POST /api/v1/apps/{uuid}/addons/{id}/restore_backup` - Restore addon from backup (PostgreSQL primary only, not available for replicas; requires `apps:operate`). Optional `backup_name` or `target_time` (not both), and boolean `to_new_cluster`: `true` restores into a new cluster and leaves the current one running, omitted or `false` replaces the current database
- `POST /api/v1/apps/{uuid}/addons/{id}/reset_database` - Reset addon database (PostgreSQL primary only, not available for replicas; requires `apps:manage`, because nothing brings the data back)

## Services

- `GET /api/v1/services` - List all services (includes `role` and `primary_addon_uuid` fields for PostgreSQL services)
- `POST /api/v1/services` - Create service (types: postgres, shared_storage. PostgreSQL supports `creation_mode`: `fresh` or `external_replica`, and `instances` [1, 3, 5, 7] for HA clusters)
- `GET /api/v1/services/{id}` - Get service details (includes `role`, `primary_addon_uuid`, and `replicas` for PostgreSQL primaries)
- `PUT /api/v1/services/{id}` - Update service (PostgreSQL supports `instances` [1, 3, 5, 7] for scaling cluster nodes, boolean `backup_enabled` and `public_access`, and a 6-field cron `scheduled_backup`). `project_id` moves the service to another project (requires `services:manage`); a service that belongs to a stack moves with the stack instead
- `DELETE /api/v1/services/{id}` - Delete service (deleting a primary cascades to all its replicas)
- `PATCH /api/v1/services/{id}/state` - Change service state (process_start/process_stop/process_restart)
- `POST /api/v1/services/{id}/rotate_password` - Rotate service password (databases and caches only, returns new password)
- `POST /api/v1/services/{id}/mount_app` - Mount an application to a service (creates storage addon on app linked to this service)
- `POST /api/v1/services/{id}/unmount_app` - Unmount an application from a service
- `POST /api/v1/services/{id}/create_replica` - Create a read replica of a PostgreSQL service (primary only, optional `cpu_size`/`ram_size` params, requires `services:operate`)
- `POST /api/v1/services/{id}/promote_replica` - Promote a read replica to a standalone PostgreSQL instance (replica only, requires `services:operate`)
- `POST /api/v1/services/{id}/promote_external` - Promote a service external replica to standalone instance or cluster by disconnecting from external source (preserves current mode, requires `services:operate`)
- `GET /api/v1/services/{id}/backups` - Get service backups (PostgreSQL primary only, not available for replicas). Answers `503` when the backup state cannot be read from the cluster: retry later, it does not mean backups are disabled. `backup_enabled` is the configured setting and changes as soon as an update is accepted; `backup_active` is what the cluster runs right now and catches up asynchronously. After enabling backups, `backup_enabled: true` with `backup_active: false` means the change is still being applied, not that it was ignored — poll until `backup_active` matches. `backup_enabled` is always `false` on the free plan
- `POST /api/v1/services/{id}/restore_backup` - Restore service from backup (PostgreSQL primary only, not available for replicas). Optional `backup_name` or `target_time` (not both), and boolean `to_new_cluster`: `true` restores into a new cluster and leaves the current one running, omitted or `false` replaces the current database
- `POST /api/v1/services/{id}/reset_database` - Reset service database (PostgreSQL primary only, not available for replicas; requires `services:manage`, because nothing brings the data back)

## What add-on and service reads return

The settings listed below come back on the read in the unit the create and update endpoints accept, so a client can read a value and send it back.

**Add-ons** (`GET /api/v1/apps/{uuid}/addons`, `GET /api/v1/apps/{uuid}/addons/{id}`):
- `ram_size` (**MiB**), `disk_size` (**GiB**) and `cpu_size` (cores) are the values `POST` and `PUT /apps/{uuid}/addons` accept. `ram_size` and `disk_size` are rounded to 6 decimal places. All three are `null` when the add-on has no quota. On a free plan, creating a PostgreSQL, MySQL or Valkey add-on stores a `cpu_size` of `0.1` whatever was sent; `PUT` stores the value sent. The raw `quota` object is still returned and holds `ram_size` and `disk_size` in **bytes**; do not send those back.
- `service_id` is the numeric ID of the service a storage add-on is mounted from, the value `POST /apps/{uuid}/addons` takes as `service_id`. `null` for add-ons not mounted from a service.
- `backup_enabled` and `scheduled_backup` are PostgreSQL only and `null` for other types. They are the effective values: `backup_enabled` is `false` on a free plan whatever was stored, otherwise the stored value, and `true` when never set. `scheduled_backup` is a 6-field cron expression, `0 0 3 * * *` when none was set.
- `role`, `primary_addon_uuid` and `replicas` are PostgreSQL only: `role` is `primary` or `replica`, `primary_addon_uuid` is set on a replica, and `replicas` lists `{uuid, name, label, state}` for a primary. `replicas` is returned by `GET /apps/{uuid}/addons/{id}` only, not by the list.
- `settings` never contains `external_config` or `replication_config`.

**Services** (`GET /api/v1/services/{id}`, and the responses of `POST /services`, `PUT /services/{id}`, `POST /services/{id}/create_replica`, `POST /services/{id}/mount_app` and `POST /services/{id}/unmount_app`). The list, `GET /api/v1/services`, returns none of the keys below:
- `resource_id` is the UUID of the assigned resource, the value `POST /services` takes. `miget_id` carries the same UUID.
- `ram_size` (**MiB**), `disk_size` (**GiB**) and `cpu_size` (cores) are read from the service's add-on quota and are `null` when the service has no add-on. `ram_size` and `disk_size` are rounded to 6 decimal places. On a free plan, creating a PostgreSQL service stores a `cpu_size` of `0.1` whatever was sent; `PUT` stores the value sent.
- `public_access` is `null` for service types without the setting (`shared_storage`) or when the service has no add-on.
- `instances` (1 for standalone, 3, 5 or 7 for an HA cluster) and `postgres_version` are PostgreSQL only and `null` for other types.
- `mount_point` is the container mount path of a `shared_storage` service and `null` for other types.
- `backup_enabled` and `scheduled_backup` are PostgreSQL only and `null` for other types or when the service has no add-on. They are the effective values, the same as on the add-on read: `backup_enabled` is `false` on a free plan whatever was stored, otherwise the stored value, and `true` when never set. `scheduled_backup` is a 6-field cron expression, `0 0 3 * * *` when none was set.
- `mounts` lists the applications mounting a `shared_storage` service, one entry per `POST /services/{id}/mount_app`, each `{app_id, addon_uuid, mount_point, sub_path, label}`, limited to apps in projects you can access. `[]` for other service types.
- The `settings` of the add-ons nested in a service never contain `external_config` or `replication_config`.

```json
{
  "uuid": "0190f2c4-6666-7000-8000-000000000006",
  "name": "postgres-mwvzq",
  "type": "postgres",
  "label": "Main database",
  "quota": {"ram_size": 536870912, "disk_size": 5368709120, "cpu_size": 0.5},
  "ram_size": 512.0,
  "disk_size": 5.0,
  "cpu_size": 0.5,
  "service_id": null,
  "backup_enabled": true,
  "scheduled_backup": "0 0 3 * * *",
  "role": "primary",
  "primary_addon_uuid": null,
  "replicas": []
}
```

```json
{
  "uuid": "0190f2c4-7777-7000-8000-000000000007",
  "resource_id": "0190f2c4-3333-7000-8000-000000000003",
  "miget_id": "0190f2c4-3333-7000-8000-000000000003",
  "service_type": "postgres",
  "ram_size": 512.0,
  "disk_size": 5.0,
  "cpu_size": 0.5,
  "public_access": false,
  "instances": 3,
  "postgres_version": "17",
  "mount_point": null,
  "mounts": [],
  "backup_enabled": true,
  "scheduled_backup": "0 0 3 * * *"
}
```

A `shared_storage` service returns `mount_point` and `mounts`, and `null` for `public_access`, `instances`, `postgres_version`, `backup_enabled` and `scheduled_backup`:

```json
{
  "service_type": "shared_storage",
  "public_access": null,
  "instances": null,
  "postgres_version": null,
  "backup_enabled": null,
  "scheduled_backup": null,
  "mount_point": "/shared-data",
  "mounts": [
    {
      "app_id": "0190f2c4-1111-7000-8000-000000000001",
      "addon_uuid": "0190f2c4-8888-7000-8000-000000000008",
      "mount_point": "/data",
      "sub_path": "media/uploads",
      "label": "Storage #1 for shared-storage-service"
    }
  ]
}
```

## Create App Addon (`POST /api/v1/apps/{uuid}/addons`)

An addon is attached to a specific application and its lifecycle is managed alongside the app.

**Common Parameters (for all addon types):**

*   **Required:**
    *   `uuid` (string, in path): The UUID of the application to attach the addon to.
    *   `type` (string): The type of addon. Must be one of `postgres`, `mysql`, `valkey`, `storage`.
    *   `label` (string): A human-readable display name for the addon.
*   **Optional:**
    *   `ram_size` (float): RAM allocation in MiB (e.g., 64, 128, 256).
    *   `disk_size` (float): Disk storage in GiB (e.g., 1, 5, 10).
    *   `cpu_size` (float): CPU allocation in cores (e.g., 0.1, 0.25, 0.5). A ceiling, not a reservation — and ignored on dev plans, where it is pinned to `0.1`.

**Connection variables.** Creating a `postgres`, `mysql`, or `valkey` addon writes **two** variables to the app, both set to the same connection string: `<ADDON_NAME>_URL`, and a generic alias — `DATABASE_URL` for `postgres`/`mysql`, `REDIS_URL` for `valkey`. The alias is skipped when the app already has a variable of that name (case-insensitive), and an existing one is never overwritten. Verify with `GET /api/v1/apps/{uuid}/vars` after creating the addon.

---

#### Addon Type: `postgres`

A PostgreSQL database addon. Supports two creation modes: fresh database or external replica.

*   **Type-specific Parameters:**
    *   `postgres_version` (string, **required**): The major version of PostgreSQL. Accepted values: `'18'`, `'17'`, `'16'`, `'15'`, `'14'`, `'13'`. Any other value is rejected with `400`.
    *   `public_access` (boolean): Enable public internet access. `true`/`false`; `'1'`/`'0'` are also accepted.
    *   `instances` (integer): Number of database instances. Allowed values: `1` (standalone, default), `3`, `5`, or `7` for a High Availability cluster.
    *   `creation_mode` (string): `'fresh'` (new empty database, default) or `'external_replica'` (replica of an external PostgreSQL database). An external replica is always created with public access.

*   **External Replica Parameters** (required when `creation_mode` is `'external_replica'`):
    *   `external_host` (string): Hostname of the external PostgreSQL source database.
    *   `external_port` (integer): Port of the external PostgreSQL source (default: 5432).
    *   `auth_type` (string): `'password'` or `'tls'`.
    *   `replication_username` (string): Username for replication connection.
    *   `replication_password` (string): Password (required when `auth_type` is `'password'`).
    *   `ca_crt` (string): CA certificate (required when `auth_type` is `'tls'`).
    *   `tls_crt` (string): TLS client certificate (required when `auth_type` is `'tls'`).
    *   `tls_key` (string): TLS client key (required when `auth_type` is `'tls'`).
    *   `s3_enabled` (boolean): `true` to enable optional S3 WAL archive fallback.
    *   `s3_endpoint`, `s3_bucket`, `s3_path`, `s3_access_key`, `s3_secret_key` (strings): S3 configuration (required when `s3_enabled` is `true`).

**Example questions:**

> "What version of PostgreSQL would you like? (18, 17, 16, 15, 14 or 13)"
> "Should this database be accessible from the public internet? (yes/no)"
> "Do you want a standalone instance or a High Availability ha_cluster? (standalone/cluster)"
> "Do you want to create a fresh database or replicate from an external PostgreSQL? (fresh/external_replica)"

---

#### Addon Type: `mysql`

A MySQL database addon.

*   **Type-specific Parameters:**
    *   `mysql_version` (string, **required**): The major version of MySQL. Accepted values: `'8.2'`, `'8.0'`. Any other value is rejected with `400`.

**Example questions:**

> "What version of MySQL would you like? (8.2 or 8.0)"

---

#### Addon Type: `valkey`

A Valkey (Redis-compatible) cache addon.

*   **Type-specific Parameters:**
    *   `valkey_version` (string, **required**): The version of Valkey. Accepted values: `'7'`, `'7.2'`. Any other value is rejected with `400`.

**Example questions:**

> "What version of Valkey would you like? (7 or 7.2)"

---

#### Addon Type: `storage`

A persistent storage volume addon.

*   **Type-specific Parameters:**
    *   `service_id` (integer): To attach an existing shared storage service, provide its ID. When given, `mount_point` and `storage_access` are inherited from that service — omit them.
    *   `mount_point` (string, **required unless `service_id` is given**): The path inside the container where the volume should be mounted (e.g., `/data`).
    *   `storage_access` (string, **required unless `service_id` is given**): The access mode. Must be one of `RWO` (ReadWriteOnce) or `RWX` (ReadWriteMany).
    *   `sub_path` (string): Mount a subdirectory of the volume instead of its root, e.g. `media/uploads` — created automatically if missing. Each app mounting the same shared volume can use a different `sub_path`, which is how several apps share one volume without seeing each other's files. Relative path only — `.` and `..` segments and backslashes are refused — and **RWX only**; sending it with `RWO` is refused with a `422`. It is equally valid on a mount attached with `service_id`, which is always RWX. The value is stored in the addon's `settings` and returned there.

**Example questions:**

> "Where should the storage be mounted inside the container? (e.g., /data)"
> "What access mode do you need? `RWO` (for a single running app instance) or `RWX` (for multiple app instances)?"

---

## Create Service (`POST /api/v1/services`)

A service is a standalone resource (e.g., database, shared storage) that can be used by multiple applications.

**Common Parameters (for all service types):**

*   **Required:**
    *   `service_type` (string): The type of service. Must be one of `postgres`, `shared_storage`.
    *   `project_id` (string): The UUID of the project this service will belong to.
    *   `label` (string): A human-readable display name for the service.
*   **Optional:**
    *   `resource_id` (string): UUID of the compute resource to provision the service on. Optional at the API level, but a service needs a resource to run on — supply this (or the legacy alias `miget_id`, deprecated) in practice.
    *   `ram_size` (float): RAM allocation in MiB.
    *   `disk_size` (float): Disk storage in GiB.
    *   `cpu_size` (float): CPU allocation in cores.

---

#### Service Type: `postgres`

A standalone PostgreSQL database service. Supports two creation modes: fresh database or external replica.

*   **Type-specific Parameters:**
    *   `postgres_version` (string, **required**): The major version of PostgreSQL. Accepted values: `'18'`, `'17'`, `'16'`, `'15'`, `'14'`, `'13'`. Any other value is rejected with `400`.
    *   `public_access` (boolean): Enable public internet access. `true`/`false`; `'1'`/`'0'` are also accepted.
    *   `environment_variables` (boolean): If `true`, writes the connection variables to the **project** the service belongs to — `<SERVICE_NAME>_URL` and `DATABASE_URL` — so every app in that project inherits them. An existing project variable of the same name is not overwritten.
    *   `instances` (integer): Number of database instances. Allowed values: `1` (standalone, default), `3`, `5`, or `7` for a High Availability cluster.
    *   `creation_mode` (string): `'fresh'` (new empty database, default) or `'external_replica'` (replica of an external PostgreSQL database).

*   **External Replica Parameters** (required when `creation_mode` is `'external_replica'`):
    *   `external_host` (string): Hostname of the external PostgreSQL source database.
    *   `external_port` (integer): Port of the external PostgreSQL source (default: 5432).
    *   `auth_type` (string): `'password'` or `'tls'`.
    *   `replication_username` (string): Username for replication connection.
    *   `replication_password` (string): Password (required when `auth_type` is `'password'`).
    *   `ca_crt` (string): CA certificate (required when `auth_type` is `'tls'`).
    *   `tls_crt` (string): TLS client certificate (required when `auth_type` is `'tls'`).
    *   `tls_key` (string): TLS client key (required when `auth_type` is `'tls'`).
    *   `s3_enabled` (boolean): `true` to enable optional S3 WAL archive fallback.
    *   `s3_endpoint`, `s3_bucket`, `s3_path`, `s3_access_key`, `s3_secret_key` (strings): S3 configuration (required when `s3_enabled` is `true`).

**Example questions:**

> "Which project should this service belong to? (Please provide the Project UUID)"
> "Which compute resource (Miget) should I provision this on? (Please provide the Miget UUID)"
> "What version of PostgreSQL would you like? (18, 17, 16, 15, 14 or 13)"
> "Should this database be publicly accessible? (yes/no)"
> "Do you want a standalone instance or a High Availability ha_cluster? (standalone/cluster)"
> "Do you want to create a fresh database or replicate from an external PostgreSQL? (fresh/external_replica)"

---

#### Service Type: `shared_storage`

A standalone shared storage volume service.

*   **Type-specific Parameters:**
    *   `mount_point` (string, **required**): The default mount path (e.g., `/shared-data`). Apps attaching to this service inherit it.
    *   `storage_access` (string): Ignored — a shared storage service is always provisioned as `RWX`.

**Example questions:**

> "Which project should this service belong to? (Please provide the Project UUID)"
> "Which compute resource (Miget) should I provision this on? (Please provide the Miget UUID)"
> "What access mode do you need? `RWO` (for a single app) or `RWX` (for multiple apps)?"

---

## Walking a customer through adding a database

1. **Ask what the application actually needs** rather than which database they
   want — "does this need to remember anything between deploys?" A cache and a
   primary store are different answers.
2. **Add-on or service?** An add-on is right when one application owns the data.
   A service is right when two applications share it, or when the data must
   outlive the application. Say which you are choosing and why; moving afterwards
   means a dump and restore.
3. **Check the resource has room.** RAM and disk come out of the resource's quota,
   and the refusal is a `422` naming what was short. On a free plan the
   application usually already holds all of it.
4. **Create it**, then read the variables back with `GET /api/v1/apps/{uuid}/vars`
   rather than assuming — the platform writes two, and one of them may have been
   skipped if the application already had a variable of that name.
5. **Redeploy the application** so it picks the variables up.

### What to suggest next

- **Backups**, for PostgreSQL — say plainly that they are not automatic on every
  plan, and check rather than promise.
- **A read replica**, if they mentioned reporting queries or analytics.
- **A rotated password**, if the connection string has ever been pasted anywhere.

## 4. Add a Database Addon

```http
# Create addon
POST /api/v1/apps/{app-uuid}/addons
{
  "type": "postgres",
  "label": "Primary database",
  "postgres_version": "17"
}

# `label` and the type's version field are REQUIRED even though the API
# marks them optional — omitting either returns 422.
#
# The addon injects TWO variables into the app, both holding the same
# connection string:
#
#   1. <ADDON_NAME>_URL — the addon's own name, upcased with dashes turned
#      into underscores. A postgres addon named "postgres-mwvzq" yields
#      POSTGRES_MWVZQ_URL.
#   2. A generic alias — DATABASE_URL for postgres and mysql, REDIS_URL
#      for valkey.
#
# The alias is created only when the app has no variable of that name yet
# (compared case-insensitively). An existing DATABASE_URL is left alone,
# so an app pointed at some other database keeps pointing at it.
#
# There are no broken-out components — no DB_HOST, DB_PORT, DB_USER.
#
# So a framework reading DATABASE_URL works with no extra step. Confirm
# with GET /apps/{uuid}/vars rather than assuming either way, and do not
# create a second DATABASE_URL — you would be duplicating a variable the
# platform already set.
```

## 8. Create a PostgreSQL Read Replica

Read replicas provide read-only copies of a PostgreSQL database for scaling read workloads. Replicas share credentials with their primary and are managed as CloudNativePG replica clusters.

```http
# For app addons:
# Step 1: Get the primary addon details
GET /api/v1/apps/{app-uuid}/addons/{addon-id}
# Verify role is "primary" and type is "postgres"

# Step 2: Create the read replica
POST /api/v1/apps/{app-uuid}/addons/{addon-id}/create_replica

# For standalone services:
# Step 1: Get the primary service details
GET /api/v1/services/{service-id}
# Verify role is "primary" and service_type is "postgres"

# Step 2: Create the read replica
POST /api/v1/services/{service-id}/create_replica
```

**Important notes about replicas:**
- Only PostgreSQL **standalone** primary addons/services can have replicas (cluster/HA databases do not support read replicas - HA clusters already provide redundancy)
- Cannot create a replica of a replica
- Replicas share credentials (database name, username, password) with their primary
- Replicas use the same resource allocation (CPU, RAM, disk) as their primary
- Replicas do **not** support backups, restore, or database reset - these operations are only available on the primary
- Replicas do **not** have their own ports or environment variables
- Public access setting is inherited from the primary and cannot be changed independently - if the primary has public access enabled, the replica's `connection_details.external` will include external connection URLs
- Deleting a primary automatically deletes all its replicas
- Replica creation is asynchronous - poll the addon/service `state` to track provisioning, testing it against the ready set in "Reading `state`" (`healthy`, `bound`, `running`) rather than a single string
- Replicas can be promoted to standalone instances using the promote endpoint - this is irreversible and the promoted instance will no longer receive updates from the primary
- The `create_replica` endpoint returns the full serialized replica entity (same shape as `GET /api/v1/apps/{uuid}/addons/{id}` or `GET /api/v1/services/{id}`), including `uuid`, `role: "replica"`, `primary_addon_uuid`, and `connection_details` - no follow-up `GET` is required to discover the new replica

**`connection_details` is empty until there is something to connect to.** It is `{}`
for shared storage, which has no connection, and `{}` for a database that is still
being provisioned - the credentials are written after the add-on row. Read it
defensively and poll `state` rather than expecting `connection_details.internal`
on the response to the create call.

**Response fields for PostgreSQL addons/services:**
- `role` (string) - `"primary"` or `"replica"` (null for non-PostgreSQL)
- `primary_addon_uuid` (string) - UUID of the primary addon (only present on replicas)
- `replicas` (array) - List of replicas with `uuid`, `name`, `label`, `state` (only present on primaries, in show endpoints)

## Create Read Replica - App Addon (`POST /api/v1/apps/{uuid}/addons/{id}/create_replica`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon ID or UUID (must be a PostgreSQL primary addon)

**Optional fields:**
- `cpu_size` (float) - CPU allocation for the replica (defaults to primary's value)
- `ram_size` (integer) - RAM allocation in megabytes (defaults to primary's value)

**Constraints:**
- Addon must be PostgreSQL type
- Addon must be a primary (not a replica itself)
- Resource must have sufficient capacity for the replica
- Requires `apps:operate` permission

**Ask only what you cannot derive:**
- "Which PostgreSQL addon should I create a replica for? (provide addon UUID)"
- "What CPU and RAM should the replica use? (defaults to primary's values)"

## Promote Replica to Standalone - App Addon (`POST /api/v1/apps/{uuid}/addons/{id}/promote_replica`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon ID or UUID (must be a PostgreSQL replica addon)

**Constraints:**
- Addon must be PostgreSQL type
- Addon must be a replica (not a primary)
- Requires `apps:operate` permission

**Ask only what you cannot derive:**
- "Which replica should I promote to a standalone instance? (provide addon UUID)"

## Promote External Replica - App Addon (`POST /api/v1/apps/{uuid}/addons/{id}/promote_external`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon ID or UUID (must be a PostgreSQL external replica addon)

**Constraints:**
- Addon must be PostgreSQL type
- Addon must be an external replica (created with `creation_mode: external_replica`)
- Promotes by disconnecting from external source
- Preserves current mode (standalone or cluster)
- Requires `apps:operate` permission

**Ask only what you cannot derive:**
- "Which external replica addon should I promote to a standalone instance? (provide addon UUID)"

## Create Read Replica - Service (`POST /api/v1/services/{id}/create_replica`)

**Required fields:**
- `id` (string, path) - Service ID (must be a PostgreSQL primary service)

**Optional fields:**
- `cpu_size` (float) - CPU allocation for the replica (defaults to primary's value)
- `ram_size` (integer) - RAM allocation in megabytes (defaults to primary's value)

**Constraints:**
- Service must be PostgreSQL type
- Service must be a primary (not a replica itself)
- Resource must have sufficient capacity for the replica
- Requires `services:operate` permission

**Ask only what you cannot derive:**
- "Which PostgreSQL service should I create a replica for? (provide service ID)"
- "What CPU and RAM should the replica use? (defaults to primary's values)"

## Promote Replica to Standalone - Service (`POST /api/v1/services/{id}/promote_replica`)

**Required fields:**
- `id` (string, path) - Service ID (must be a PostgreSQL replica service)

**Constraints:**
- Service must be PostgreSQL type
- Service must be a replica (not a primary)
- Requires `services:operate` permission

**Ask only what you cannot derive:**
- "Which replica service should I promote to a standalone instance? (provide service ID)"

## Promote External Replica - Service (`POST /api/v1/services/{id}/promote_external`)

**Required fields:**
- `id` (string, path) - Service ID (must be a PostgreSQL external replica service)

**Constraints:**
- Service must be PostgreSQL type
- Service must be an external replica (created with `creation_mode: external_replica`)
- Promotes by disconnecting from external source
- Preserves current mode (standalone or cluster)
- Requires `services:operate` permission

**Ask only what you cannot derive:**
- "Which external replica service should I promote to a standalone instance? (provide service ID)"

## Rotate Addon Password (`POST /api/v1/apps/{uuid}/addons/{id}/rotate_password`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon UUID

**Constraints:**
- Password rotation only supported for certain addon types (databases like PostgreSQL, MySQL, and caches like Valkey)
- Requires `apps:operate` permission
- New password will be returned in response - you must update ENV variables manually

**Ask only what you cannot derive:**
- "Which addon should I rotate the password for? (provide addon UUID)"

## Rotate Service Password (`POST /api/v1/services/{id}/rotate_password`)

**Required fields:**
- `id` (string, path) - Service ID

**Constraints:**
- Only supported for database/cache services (PostgreSQL, MySQL, Valkey)
- Requires `services:operate` permission
- New password returned in response - update ENV variables manually

## Change Addon State (`PATCH /api/v1/apps/{uuid}/addons/{id}/state`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Addon ID or UUID
- `state` (string) - Target state: `process_start`, `process_stop`, or `process_restart`

## Change Service State (`PATCH /api/v1/services/{id}/state`)

**Required fields:**
- `id` (string, path) - Service ID
- `state` (string) - Target state: `process_start`, `process_stop`, or `process_restart`

## Mount App to Service (`POST /api/v1/services/{id}/mount_app`)

Creates a storage addon on the app linked to this service.

**Required fields:**
- `id` (string, path) - Service UUID
- `app_id` (string) - UUID of the application to mount

**Optional fields:**
- `mount_point` (string) - Container mount path (e.g., /data)
- `sub_path` (string) - Subdirectory of the shared volume to mount instead of its root, e.g. `media/uploads`; created if missing. Each mounted app can use a different `sub_path` to keep its files apart on the same shared volume. Relative path only — `.` and `..` segments and backslashes are refused — and RWX (shared) storage only.
- `label` (string) - Display label for the mounted addon

**Read the mount back from `settings`.** The storage addon's `settings` carries
`mount_point`, `access_type` and `sub_path`, so `GET /api/v1/apps/{uuid}/addons/{id}`
tells you where inside the shared volume an application is actually writing. That
is the only way to check it — nothing else reports it.

**A service mounts to a given application once.** Calling `mount_app` again for
the same pair is refused with **422** rather than moving the mount, because
`unmount_app` matches a single addon by service and could not tell two of them
apart. To change where an application mounts a volume, unmount it and mount it
again at the new path.

## Unmount App from Service (`POST /api/v1/services/{id}/unmount_app`)

**Required fields:**
- `id` (string, path) - Service UUID
- `app_id` (string) - UUID of the application to unmount

---
