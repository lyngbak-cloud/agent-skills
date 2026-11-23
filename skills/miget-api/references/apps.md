# Applications

Everything that belongs to one application: creating it, the six ways to get code
into it, deploying, watching a deployment, and the settings that hang off it —
variables, secret files, domains, ports, cron jobs, health checks, scaling,
security, preview environments and cloning.

Read this before the first `POST /api/v1/apps` of a session. The required fields
are not repeated in SKILL.md, and the deployment method decides which
`deployment_config` fields are mandatory — posting without checking returns `400`
or `422` and you will be guessing at which of a dozen fields it meant.

## Contents

- [Endpoints](#endpoints)
- [Creating an application](#create-application-post-apiv1apps)
- [Walking a customer through their first deploy](#walking-a-customer-through-their-first-deploy)
- [Deploying and watching it](#1-create-and-deploy-an-application)
- [Variables, files, domains, ports, cron](#3-manage-environment-variables)
- [Settings: security, health checks, scaling, cloning](#update-security-settings-put-apiv1appsuuidsecurity)
- [Worked examples](#example-complete-application-setup-github)

## What an application is

## Applications (Apps)

An **Application** is a deployable service (web app, API, worker, etc.).

- Apps are deployed to a **Resource** (Miget)
- Apps can have multiple **deployment methods**:
  - `git_push` - Push to Miget-hosted Git remote
  - `public_git` - Public Git repository
  - `github` - GitHub repository (via GitHub App)
  - `container_registry` - Pre-built container image from a registry (Docker Hub, GHCR, etc.). Not available on a free-plan resource
  - `parent_image` - Inherit image from parent app
  - `kamal` - Deploy from local machine using Kamal
- Apps belong to a **Project** (for organization)
- Apps can have **addons** (databases, caches, storage)
- Apps can have **cronjobs** (scheduled tasks)
- Apps can have **workers**: the processes their Procfile runs besides the web (`sidekiq`, `clock`, a queue worker). `GET /apps` and `GET /apps/{uuid}` list them under `workers`, each with `name`, `state` (`running`, `stopped` at zero replicas, or `failed`), `replicas` and `quota` (the same shape and units as the app's). They come from the code: no endpoint adds, removes or resizes one, and `PUT /api/v1/apps/{uuid}/workers/{name}` scales one (see "Scale a Worker" below). app.json's `formation` sets a worker's replicas on every deploy (`quantity`), and its size when it names one (`size`): a size code as `GET /api/v1/workspace/pay_per_use/sizes` lists it (`hobby-1c-512m`), or a Heroku dyno size: `eco`, `basic`, `standard-1x` (and the older `free`, `hobby`, `1x`) run at `hobby-1c-512m`, `standard-2x` (`2x`) at `hobby-1c-1g`, `private-s`/`shield-s` at `pro-1c-2g`, `performance-m`/`private-m`/`shield-m`/`px` at `pro-2c-4g`, `*-l` at `pro-8c-16g`, `*-l-ram` at `pro-16c-32g`, `*-xl` at `pro-32c-64g`, and a Fir `dyno-Nc-Mgb` at the smallest Pro size with at least N CPU and M GB. A worker with no size runs at the app's size; one the app's tier has no match for (a Pro code, performance or private dyno on a Hobby app, a Hobby code or shared dyno on a Pro app, any `*-2xl`) runs at the app's size and at most one replica, and the deployment says so. The `web` entry's `size` sets the app's own size the same way (not on free compute), over what the panel or `PUT /api/v1/apps/{uuid}` set: app.json wins on every deploy. Every size is held to what the app may run (the room on its resource, or the largest Usage Billing size), and `quota` reports what the build actually started. On a resource, workers come after everything else: its `available_ram_size` leaves them out. On Usage Billing, a worker is billed like its app, at its own size, for the replicas it runs.
- Apps can have **domains** (custom domains)
- Apps can have **environment variables** (vars)
- Apps can have **ports** (exposed ports). Port `5000` is fixed: HTTP traffic on the app's `*.migetapp.com` URL is always served from port `5000` — the app must listen on `5000`, and this port cannot be removed or changed. Additional TCP/UDP ports can be added for custom protocols; they are **private by default** and can be exposed publicly via the expose endpoint (see "Manage App Ports" below, and https://docs.miget.com/networking/ports for the full list of supported ports).
- Apps can be **public or private** (`private_access`): a private app has no public ingress and is reachable only inside the workspace network. Settable on create/update (default `false`); returned in the app response.
- Apps have an **internal URL** for app-to-app and addon connections, returned as `internal_url` on the app response in the form `<service_name>.<resource-name>.<region-code>.migetapp.internal:5000`. Traffic from other apps requires `allow_connections: true` on the **destination** app (default `false`, set via `PUT /apps/{uuid}/security`); once enabled, other applications on the same resource (miget) can reach it at its `internal_url`.
- Apps also have a **public URL**, returned as `public_url` on the app response in the form `https://<name>.<region-code>.migetapp.com`. The region comes from the compute resource the app runs on, or from `region_code` for an app on Usage Billing or in the workspace pool; every app has one. It is built from `name`, not `label` or `service_name` — and because the server appends a random suffix to `name` at creation, the only reliable way to learn an app's URL is to read `public_url` back from the response. An app with `private_access: true` has no public ingress, so its `public_url` will not answer. Custom domains are **not** included here; they are listed separately under `GET /apps/{uuid}/domains`, which returns `[]` for an app that only has its platform URL.
- Resource limits are returned as `ram_size` (**MiB**, rounded to 6 decimal places) and `cpu_size` (cores), per instance, the same units `POST` and `PUT /apps` accept. Both are `null` when the app has no quota. The raw `quota` object is still returned, with its own `ram_size` in **bytes** (e.g. `134217728` = 128 MiB), so do not send `quota.ram_size` back as `ram_size`.
- The app response also returns `basic_auth_enabled` (whether HTTP Basic Auth is enforced at the ingress) and `basic_auth_username` (`null` when none is set). The Basic Auth password is **never** returned.
- `resource_id` is the UUID of the assigned compute resource, the value `POST /apps` takes; `null` unless `billing` is `"resource"`. `vpc_uuid` and `vpc_subnet_uuid` identify the VPC and subnet the app is attached to, `null` when it is not attached.
- `project_variables_enabled` says whether the app also receives its project's environment variables. `project_secret_files_enabled` says whether it also mounts its project's secret files; it is changed with `PUT /apps/{uuid}/secret_files/project_files_enabled`.
- `health_checks` returns every health check setting under the names `PUT /apps/{uuid}/health_checks` takes, so a client can send back what it read. A path or number that was never set is returned as `null` and a flag that was never set as `false`.
- `scaling_profile` returns the settings under the names `PUT /apps/{uuid}/scaling_profile` takes (`replicas`, `auto_scaling_enabled`, `auto_min_replicas`, `auto_max_replicas`, `scaling_start_time`, `scaling_end_time`, `cpu_threshold`, `memory_threshold`). `scaling_start_time` and `scaling_end_time` are `HH:MM` strings, `null` when no scheduled window is set. An app with no scaling profile runs one instance: `replicas` is `1`, `auto_scaling_enabled` is `false` and the other keys are `null`.
- `deployment_config.credential_id` (for `container_registry`, `github` and `public_git`) is the UUID of the stored credential, the value `PUT /apps/{uuid}/deployment` accepts, `null` when none is set. `container_registry` also returns `command` and `args`; `github` also returns `dockerfile_path` and `build_context`.
- Every app automatically gets **monitoring** — Grafana dashboards, metrics, and logs, with Prometheus/Loki-compatible query APIs at `metrics.miget.com`. Runtime metrics and app logs are **not** on the REST API; see the Monitoring & Observability section.

## Endpoints

## Applications

- `GET /api/v1/apps` - List all applications
- `POST /api/v1/apps` - Create new application. Boolean `auto_sleep` (Usage Billing and the workspace pool only, requires `apps:operate`) creates it already set to sleep when idle; it is refused with `422` on a resource, on free compute and with `private_access`
- `GET /api/v1/apps/{uuid}` - Get application details (returns the settings described above, see the example below; includes `deployment_method` and `deployment_config` with method-specific fields; for Kamal apps this includes `registry_password`, `registry_hostname`, `registry_username`, `registry_image`, and `ssh_keys`; also includes a nested `region` object with `id`, `name`, and `code`, plus a `private_access` boolean)
- `PUT /api/v1/apps/{uuid}` - Update application. Boolean `auto_sleep` (Usage Billing and the workspace pool only, requires `apps:operate`) makes the app sleep when idle: after 15 minutes without traffic it scales to zero, its workers included, and the next request wakes it, answered with `503` while it starts. While it sleeps its CPU and RAM are not billed; its volume still is. Its PostgreSQL add-ons sleep with it unless set to keep running. `auto_sleep` is refused with `422` on a resource and on free compute, which always sleeps; `true` is refused on a private application (`private_access`), which has no public route to wake it. `GET` returns `auto_sleep` as whether the app sleeps when idle, `true` on free compute
- `DELETE /api/v1/apps/{uuid}` - Delete application
- `PUT /api/v1/apps/{uuid}/security` - Update security settings (network connectivity, Basic Authentication)
- `PATCH /api/v1/apps/{uuid}/state` - Change app state (schedule_start/schedule_stop/schedule_restart)
- `POST /api/v1/apps/{uuid}/clone` - Clone an application. Copies nothing by default beyond the app's own settings — env vars, secret files, scaling, health checks, security, add-ons and cronjobs are each opt-in. Requires `apps:manage`. The clone runs on a resource, or on Usage Billing or in the workspace pool like a new app. A resource assigned to projects other than the target is refused with `422`. See the runbook under Endpoint Reference
- `PUT /api/v1/apps/{uuid}/deployment` - Update deployment method and configuration (switch methods, update Kamal SSH keys). `deployment_config_attributes` is a **patch**: fields you omit keep their stored value, and a field sent as `""` is cleared. Sending a *different* `deployment_method` builds the config from scratch, so supply every field that method needs.
- `POST /api/v1/apps/{uuid}/deploy` - Trigger deployment (optional: custom_tag, commit_sha, branch). Not used for Kamal apps. Returns `409 Conflict` if a deployment is already in progress — poll `GET /apps/{uuid}/deployments` and retry once it settles. On a `github` app, a `commit_sha` that does not exist in the configured repository is rejected with `422` before any build starts, so push the commit first and pass a SHA from the same repository the app is configured with (a SHA from a fork or a squashed/force-pushed branch will not resolve). Other deployment methods do not check the SHA.
- `PUT /api/v1/apps/{uuid}/health_checks` - Update health check probes (liveness, readiness, startup)
- `PUT /api/v1/apps/{uuid}/scaling_profile` - Update scaling profile (replicas, auto-scaling, thresholds, and for an app on Usage Billing a monthly budget). Not available on free plan.
- `PUT /api/v1/apps/{uuid}/workers/{name}` - Scale one worker on its own (replicas or auto-scaling); the web and the other workers keep theirs. Not available on free plan.
- `GET /api/v1/apps/{uuid}/deployments` - List deployments
- `GET /api/v1/apps/{uuid}/deployments/{id}/logs` - Get build logs
- `GET /api/v1/apps/{uuid}/activity` - Get paginated activity feed (deployments, config changes, audit events). Query params: `page`, `limit`. Returns an envelope `{ "activities": [{ action, description, resource, actor, timestamp, source }], "pagination": { page, limit, total } }`.

### Example: `GET /api/v1/apps/{uuid}` (settings keys)

```json
{
  "uuid": "0190f2c4-1111-7000-8000-000000000001",
  "name": "acme-api-x7k2p",
  "label": "Acme API",
  "state": "running",
  "deployment_method": "github",
  "deployment_config": {
    "type": "github",
    "repository": "acme/api",
    "branch": "main",
    "auto_deploy_enabled": true,
    "credential_id": "0190f2c4-2222-7000-8000-000000000002",
    "dockerfile_path": "Dockerfile",
    "build_context": "."
  },
  "basic_auth_enabled": true,
  "basic_auth_username": "staging",
  "quota": {"ram_size": 268435456, "cpu_size": 0.25},
  "ram_size": 256.0,
  "cpu_size": 0.25,
  "resource_id": "0190f2c4-3333-7000-8000-000000000003",
  "vpc_uuid": "0190f2c4-4444-7000-8000-000000000004",
  "vpc_subnet_uuid": "0190f2c4-5555-7000-8000-000000000005",
  "project_variables_enabled": true,
  "project_secret_files_enabled": false,
  "health_checks": {
    "liveness_probe_enabled": true,
    "liveness_probe_path": "/healthz",
    "liveness_probe_initial_delay_seconds": 10,
    "liveness_probe_timeout_seconds": 2,
    "liveness_probe_period_seconds": 10,
    "liveness_probe_failure_threshold": 3,
    "liveness_in_app_failure_notification_enabled": true,
    "liveness_email_failure_notification_enabled": false,
    "readiness_probe_enabled": false,
    "readiness_probe_path": null,
    "startup_probe_enabled": false,
    "startup_probe_path": null
  },
  "scaling_profile": {
    "replicas": 2,
    "auto_scaling_enabled": true,
    "auto_min_replicas": 2,
    "auto_max_replicas": 6,
    "scaling_start_time": "08:00",
    "scaling_end_time": "20:00",
    "cpu_threshold": 70,
    "memory_threshold": 80
  }
}
```

The `health_checks` object above is shortened: it always carries all 24 keys (readiness and startup have the same `_probe_initial_delay_seconds`, `_probe_timeout_seconds`, `_probe_period_seconds`, `_probe_failure_threshold` and `_in_app` / `_email` `_failure_notification_enabled` keys as liveness).

## App Deployments

- `GET /api/v1/apps/{uuid}/deployments` - List deployments (optional filters: `status` (pending/running/completed/failed/cancelling/cancelled), `period` (7days/30days/90days/all)). Each record includes `commit_sha`, `commit_message`, and `branch` for git-based deployment methods (null otherwise).
- `GET /api/v1/apps/{uuid}/deployments/{id}` - Get deployment details
- `GET /api/v1/apps/{uuid}/deployments/{id}/logs` - Get build logs (text/plain, available after deployment completes)
- `GET /api/v1/apps/{uuid}/deployments/{id}/stream_logs` - Stream logs in real-time (SSE, text/event-stream, for running deployments)
- `POST /api/v1/apps/{uuid}/deployments/{id}/cancel` - Cancel running deployment (only deployments in 'running' state can be cancelled)
- `POST /api/v1/apps/{uuid}/deployments/{id}/rollback` - Rollback to a previous deployment (only rollbackable deployments can be rolled back - must have image URL, same deployment method as app, and not be running/failed)

## App Environment Variables

Variables are addressed by `key`. The `id` in the list response is not accepted by any endpoint — there is no `/vars/{id}`.

- `GET /api/v1/apps/{uuid}/vars` - List app variables
- `POST /api/v1/apps/{uuid}/vars` - Create a variable. A key the app already has is refused with `422`
- `PUT /api/v1/apps/{uuid}/vars` - Set a variable (identified by `key` in body): overwrites it if it exists, creates it if not. Use this whenever you are not certain the key is new. Optionally carries `project_variables_enabled` alongside the change
- `DELETE /api/v1/apps/{uuid}/vars` - Delete variable (identified by `key` in body)
- `PUT /api/v1/apps/{uuid}/vars/project_variables_enabled` - Toggle whether the app also receives its **project's** variables (body: `enabled`). Use this to flip the switch on its own; the app's own variables are untouched

## App Secret Files

For config a container needs **on disk** rather than in the environment — `serviceAccount.json`, `.npmrc`, certificates, a `config.yaml`. Reach for these when the content is multi-line, is a file format, or is what a library expects at a path.

**`text` is write-only.** You send it; no endpoint ever returns it. Contents are encrypted at rest, so plan for the fact that you cannot read a file back to diff it — keep the source of truth on your side. The **server assigns `name`** on creation (suffixed, like every other Miget name); that `name` is the identifier for the show, update and delete endpoints. Changes reach running containers on the **next deployment**, not immediately.

- `GET /api/v1/apps/{uuid}/secret_files` - List the app's own secret files (metadata only). Project-level files inherited by this app are **not** included; read those from the project endpoint
- `POST /api/v1/apps/{uuid}/secret_files` - Create one (body: `filename`, `text`). `filename` is the in-container path and must be unique within the app; a duplicate is a **422**. Returns **201** with the assigned `name`
- `GET /api/v1/apps/{uuid}/secret_files/{name}` - Get one file's metadata
- `PUT /api/v1/apps/{uuid}/secret_files/{name}` - Update `filename`, `text`, or both. Omitted fields keep their value; sending neither is a **400**
- `DELETE /api/v1/apps/{uuid}/secret_files/{name}` - Remove it
- `PUT /api/v1/apps/{uuid}/secret_files/project_files_enabled` - Toggle whether the app also mounts its **project's** secret files (body: `enabled`). Creating a project secret file does nothing for an app until this is on

## App Domains

- `GET /api/v1/apps/{uuid}/domains` - List app domains
- `POST /api/v1/apps/{uuid}/domains` - Add domain
- `GET /api/v1/apps/{uuid}/domains/{domain_uuid}` - Get domain details (returns `verification_status`, `verification_token`, `dns_target`)
- `PUT /api/v1/apps/{uuid}/domains/{domain_uuid}` - Update domain
- `DELETE /api/v1/apps/{uuid}/domains/{domain_uuid}` - Remove domain
- `POST /api/v1/apps/{uuid}/domains/{domain_uuid}/verify` - Trigger DNS verification. Caller is expected to have already published `TXT _migetapp-verify.<domain> = <verification_token>` (returned by GET). The check runs async (Fibonacci backoff up to 60min); poll the GET endpoint until `verification_status` becomes `verified` and `dns_target` is populated.

## App Ports

- `GET /api/v1/apps/{uuid}/ports` - List all ports for an app
- `POST /api/v1/apps/{uuid}/ports` - Create a new port (requires apps:operate, not available on free plan)
- `GET /api/v1/apps/{uuid}/ports/{port_id}` - Get port details
- `DELETE /api/v1/apps/{uuid}/ports/{port_id}` - Delete a port
- `PATCH /api/v1/apps/{uuid}/ports/{port_id}/expose_publicly` - Expose a port publicly (requires apps:operate, not available on free plan)
- `PATCH /api/v1/apps/{uuid}/ports/{port_id}/make_private` - Make a port private (requires apps:operate)

## App Cronjobs

- `GET /api/v1/apps/{uuid}/cronjobs` - List cronjobs
- `GET /api/v1/apps/{uuid}/cronjobs/sizes` - The sizes a cron job of this app can run at: `id` (the size code), `tier`, `cpu`, `ram_mb` and `cents_per_minute`. Requires `apps:view`
- `POST /api/v1/apps/{uuid}/cronjobs` - Create cronjob
- `GET /api/v1/apps/{uuid}/cronjobs/{id}` - Get cronjob details
- `PUT /api/v1/apps/{uuid}/cronjobs/{id}` - Update cronjob (**only `label`, `command`, `size`, `max_run_seconds` and `overage` are updatable**; the schedule cannot be changed via PUT — DELETE and recreate the cronjob to reschedule)
- `DELETE /api/v1/apps/{uuid}/cronjobs/{id}` - Delete cronjob
- `POST /api/v1/apps/{uuid}/cronjobs/{id}/pause` - Pause a cronjob: it starts no new run, a run already under way finishes, and while paused it is not billed and holds nothing in its resource. Requires `apps:operate`
- `POST /api/v1/apps/{uuid}/cronjobs/{id}/resume` - Resume a paused cronjob. A cron job set to pause past its resource's cron minutes, while they are used up for the cycle, is refused with `422` and resumes by itself at the next renewal; set its `overage` to `bill` (`PUT .../cronjobs/{id}`) to run it now. Requires `apps:operate`
- `GET /api/v1/apps/{uuid}/cronjobs/{id}/stream_logs` - Stream the most recent run's logs in real-time (SSE, text/event-stream; returns 404 until the job has run at least once)
- `GET /api/v1/apps/{uuid}/cronjobs/{id}/runs` - The cron job's runs of the last 30 days, newest first, a page at a time: `{"runs": [...], "pagination": {"page", "limit", "total"}}`, with optional `page` (from 1) and `limit` (1 to 100, default 20). Each run has `job_name`, `status` (`running`, `succeeded`, `failed`), `started_at`, `finished_at` and `duration_seconds` (null while running), `exit_code`, and `reason` and `message` (for a failed run the Job's `DeadlineExceeded` or `BackoffLimitExceeded`, else the container's, such as `OOMKilled` or `Error`; a run that succeeded usually carries `Completed`). Runs are kept for 30 days. Requires `apps:view`

## App Preview Environments

Ephemeral clones of an app, one per GitHub pull request or branch. **Only for `github` apps**, and **never for stack-managed apps** — those return **403** with `Preview environments are not available for stack-managed applications.`

**There is no create endpoint.** Environments appear when GitHub webhooks fire against a config whose triggers match. Your job is the config; the platform does the rest.

- `GET /api/v1/apps/{uuid}/preview_environments` - List, newest first. Optional `status` filter: `creating`, `active`, `updating`, `failed`, `destroying`. Each entry carries `app_uuid` and `url` of the cloned app, plus the `branch` and `commit_sha` it runs
- `GET /api/v1/apps/{uuid}/preview_environments/{id}` - Get one. `{id}` is the numeric `id` from the list, not a UUID
- `DELETE /api/v1/apps/{uuid}/preview_environments/{id}` - Schedule teardown. Returns **202**; the entry stays listed as `destroying` until it finishes
- `POST /api/v1/apps/{uuid}/preview_environments/{id}/redeploy` - Redeploy the commit it already tracks. Returns **202**. Use it to retry a `failed` environment — new commits deploy on their own while `auto_deploy_on_push` is on
- `GET /api/v1/apps/{uuid}/preview_environments/config` - Read the config. **404 when the app has never been configured** — that is the normal empty state, not an error; PUT to create it
- `PUT /api/v1/apps/{uuid}/preview_environments/config` - Create the config the first time, update it after. Omitted fields keep their value. While `enabled` is true, a resource assigned to projects other than the target — `project_id`, or the app's own when omitted — is refused with `422`, whether it is `resource_id` or the app's own resource in `parent` mode. A config saved before its resource was assigned to another project is turned off (`enabled: false`) at the next pull request, and the reason is sent by email and in-app notification — pick another resource or project, then enable it again. An app on a resource can also run them without one: `resource_mode: pay_per_use` (`workspace_pool` on an enterprise contract) puts each preview environment in the app's region, billed on its own while it runs. Without a resource a preview environment runs at the size set by `size` (a code from `GET /api/v1/workspace/pay_per_use/sizes`, which carries its CPU class; an empty string goes back to the app's size) or on the CPU class set by `tier` — the app's own until you set them, so a preview of a dedicated, large production app need not be either. The add-ons it clones keep the app's sizes and run on the preview's CPU class. A size the class does not offer is a `422`. An app on Usage Billing (or in the workspace pool) takes only `parent`, which runs its preview environments the same way; any other mode is a `422`. Saving with `enabled: true` and a placement without a resource sets it up at once: `422` when Usage Billing is turned off for the workspace or when the app or an add-on it clones needs more RAM or disk than a workload without a resource can have, and `422` with `checkout_url` when the workspace has to pay the $5 Usage Billing monthly minimum first — send the user there, then PUT again. A workspace that no longer holds the minimum when a pull request arrives has the config turned off with that reason, since nobody is there to complete a checkout: enable it again to start the minimum. Turning on preview environments on Usage Billing, or moving enabled ones onto `pay_per_use`, takes `apps:manage` (`403` otherwise); turning them off and editing them afterwards takes `apps:operate`. When a workspace admin turns Usage Billing off, preview environments on Usage Billing are turned off with that reason

## Container Registry Credentials

Workspace-level credentials for pulling images from private registries. The returned `uuid` is what you pass as `deployment_config.credential_id` for `container_registry` (and `github`/`public_git`) deployments. The `token` is encrypted at rest and **never returned** by the API.

- `GET /api/v1/container_registry_credentials` - List credentials
- `POST /api/v1/container_registry_credentials` - Create a credential
- `GET /api/v1/container_registry_credentials/{uuid}` - Get credential details
- `PUT /api/v1/container_registry_credentials/{uuid}` - Update a credential (rotate token, change registry/username)
- `DELETE /api/v1/container_registry_credentials/{uuid}` - Delete a credential

## Git Credentials

Workspace-level Git credentials (GitHub App installs / personal access tokens) used to clone private repositories. The returned `uuid` is what you pass as `credential_id` when creating a **stack** (`POST /api/v1/stacks`) or a `github`/`public_git` **app**. Read-only over the API; the access token is encrypted at rest and **never returned**. (Creating a GitHub App credential is a browser-based install flow, done in the dashboard.)

- `GET /api/v1/git_credentials` - List git credentials (returns `uuid`, `name`, `provider`, `installation_id`)
- `GET /api/v1/git_credentials/{uuid}` - Get credential details

## Create Application (`POST /api/v1/apps`)

**Required fields:**
- `name` (string) - Service name seed (lowercase, alphanumeric with hyphens). **The server appends a random suffix**, so the app you get back is named `my-api-x7k2p`, not `my-api`. Never build a URL or a Git remote from the name you sent — read `name` and `public_url` back from the create response. The unsuffixed form is kept separately as `service_name`, and `internal_url` is the **only** place it is used; every other identifier — `public_url`, the `git_push` repository path, the addon's `<ADDON_NAME>_URL` key — is built from the suffixed `name`. Reaching for `service_name` anywhere else produces a path that does not resolve. The suffix costs 6 characters and is applied *before* the 40-character limit is checked, so keep what you send to 34 characters or fewer — otherwise you get "Name is too long (maximum is 40 characters)" for a name that looked well under it. **`service_name` gets no suffix, and it must be unique across the whole workspace** — two apps seeded `api` collide even on different resources and in different projects, and the second is refused with a 422 naming the codename as already taken. Disambiguate the seed (`shop-api`, `blog-api`) rather than relying on the suffix, which protects `name` but not `service_name`.
- `label` (string) - Human-readable display name
- `project_id` (string) - UUID of the project to create the application in (get from `GET /api/v1/projects`)
- `resource_id` (string) - UUID of the compute resource (Miget) to assign (get from `GET /api/v1/resources`). The app's region is derived from this resource. **Not required when `billing` is `"pay_per_use"` or `"workspace_pool"`** — see "Usage Billing" below. Otherwise, leaving it out is refused with a `422`.
- `builder` (string) - Build strategy: `"auto"`, `"dockerfile"`, or `"custom"`. `"custom"` additionally requires `language` and `build_command` in `deployment_config` (see "Build Settings for `public_git` and `github`").

**Optional but important:**
- `ram_size` (float) - RAM allocation in MiB. On a resource only
- `cpu_size` (float) - CPU allocation in cores. On a resource only
- `size` (string) - On Usage Billing or in the workspace pool only, instead of `ram_size` and `cpu_size` (see "Usage Billing" below)
- `deployment_method` (string) - `"git_push"`, `"public_git"`, `"github"`, `"container_registry"`, `"parent_image"`, or `"kamal"`. Note: the enum value is `container_registry` (not `docker_registry`). `container_registry` requires a paid resource — on a free-plan resource every write is rejected with `422`: creating an app with it, switching an existing app onto it, and changing the configuration (the image tag included) of an app already deployed from a registry.
- `deployment_config` (object) - Configuration specific to the chosen deployment method (see table below)
- `app_vars_attributes` (array) - Environment variables to set at creation (array of `{key, value}` objects)
- `private_access` (boolean) - Restrict the app to private access only — no public ingress, reachable only inside the workspace network (default `false`). Also accepted on `PUT /api/v1/apps/{uuid}`.
- `vpc_uuid` (string) - Join a VPC at creation instead of attaching afterwards, which saves the restart an attach costs. The VPC's default subnet is used. The VPC must be in the same region as `resource_id`. Refused with a `422` for `size: "free"`: free compute cannot join a private network.
- `vpc_subnet_uuid` (string) - Join this subnet specifically, when the VPC has more than one. It implies its VPC, so `vpc_uuid` can be left out.
- `vpc_static_ip` (boolean) - Give the app a fixed address on the subnet it joins. Ignored unless `vpc_uuid` or `vpc_subnet_uuid` is given. Deploys then replace the pod instead of rolling it, so the app is briefly down on every deploy, and the app must keep a single replica without autoscaling. The address is returned by `GET /api/v1/vpcs/{uuid}/attachments`. See `references/networking.md`.

#### Usage Billing

Instead of `resource_id`, an app can run on Usage Billing: without a resource, billed for the CPU, RAM and storage it reserves while it runs (rates and the $5 monthly minimum are in `references/workspace.md`). Send:

- `billing` (string) - `"pay_per_use"`. `resource_id` is then ignored. (`"resource"`, the default, runs it on `resource_id`.)
- `region_code` (string) - Where to run, e.g. `eu-east-1` (get from `GET /api/v1/regions`). Defaults to a region the workspace already runs in, or in a workspace with nothing placed yet to the first active region: send it when the region matters.
- `size` (string) - A size code from `GET /api/v1/workspace/pay_per_use/sizes`, e.g. `hobby-1c-2g`: the CPU and RAM each replica reserves and is billed for, and its CPU class (`hobby-` shared, `pro-` dedicated). Leaving it out takes the smallest listed size with at least half a core. A size that is not listed is refused with a `422` naming the ones that are. **`cpu_size` and `ram_size` are refused with a `422` for an app on Usage Billing** — on create and on `PUT /api/v1/apps/{uuid}`, which resizes it by `size` too.
- `tier` (string) - `"hobby"` (a shared slice of CPU) or `"pro"` (dedicated cores), for when no `size` names it; a `size` of the other class is refused with a `422`. **Hobby and Pro workloads run apart, and a workload cannot move between them or onto a resource afterwards**, so the choice is permanent for that app.

Pick `size` deliberately: it is what is billed, on every replica. An app on Usage Billing scales up to the workspace's `replicas_per_process` limit, 20 on Team and set by the workspace plan unless raised (`GET /api/v1/workspace/limits`), in `replicas`, `auto_min_replicas` and `auto_max_replicas` of `PUT /api/v1/apps/{uuid}/scaling_profile`, each replica held to the largest standard plan of its CPU class and billed on its own; more is refused with `422`. The first workload on Usage Billing in a region may sit in `pending` a few moments longer than usual before it starts. If the workspace has no subscription to put the monthly minimum on, the request is refused with `422 {"errors": "...", "checkout_url": "..."}` and no app is created; send the person to `checkout_url`, then repeat the request. On an enterprise contract, or in a workspace that turned Usage Billing off, `billing: "pay_per_use"` is refused with a `422`.

The response carries `billing: "pay_per_use"` and `miget: null`.

#### In the workspace pool

On an enterprise contract send `billing: "workspace_pool"` instead, with the same `region_code`, `size` and `tier` (the sizes endpoint returns no prices there). Nothing is billed by the hour; the app counts against the contract's capacity, and a request the pool cannot hold is refused with `422` naming what ran out. The response carries `billing: "workspace_pool"` and `miget: null`. See "Workspace pool" in `references/workspace.md`.

#### Deployment Configuration by Method

Each `deployment_method` requires different fields in `deployment_config`:

**`git_push`** - Deploy by pushing code to a Miget-hosted Git remote. No `deployment_config` fields are required at creation.

```json
{
  "deployment_method": "git_push",
  "deployment_config": {
    "dockerfile_path": "./Dockerfile",
    "build_context": "."
  }
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `dockerfile_path` | string | `"./Dockerfile"` | Path to Dockerfile in the repository |
| `build_context` | string | `"."` | Docker build context directory |

The response's `deployment_config` of a `git_push` app also carries `ssh_url` and `https_url`, the two remotes to push to (read-only).

**Before you choose `git_push` at all.** It is the method for code with no remote. If the repository already has one, `github` or `public_git` deploys from it with no credential handling, and `github` additionally gives auto-deploy and review apps — see "Choosing Defaults" in `SKILL.md`. Check `git remote -v` first.

**Helping the user deploy a `git_push` app — SSH first.** There are two ways to authenticate, and they are not equally convenient. **Prefer SSH.** It uses the SSH key already on the user's account, needs no Git token, and every step is doable over the API. Fall back to HTTPS only when SSH genuinely will not work.

*Read the remote from `GET /api/v1/apps/{uuid}`: `deployment_config.ssh_url` and `deployment_config.https_url` are the two remotes of a `git_push` app, ready to use.* An app on Usage Billing or in the workspace pool has no resource (`miget` is `null`), so these two fields are the only place its remote is given; do not try to build it yourself. They are built from `region.code`, the namespace the app runs in and its `name`. **The repository path is the app's `name`, with its random suffix — `wall-pfhqv`, not `wall`.** Do not substitute `service_name` or `label`, and do not strip the suffix because it looks like noise: `service_name` is the unsuffixed form and belongs to `internal_url` only. Pushing to the unsuffixed path fails to authenticate against a repository that does not exist. The first path segment is the resource or namespace name (`migetmxq`), never the resource's display label.

**Option A — SSH (default).**

1. **Check the account for a key:** `GET /api/v1/users/me/ssh_keys`. If one is registered, there is nothing to set up — go to step 3.
2. **If the list is empty, register one.** A public key is not a secret, so you can read the user's own and send it: look for `~/.ssh/id_ed25519.pub` (or `id_rsa.pub`), and `POST /api/v1/users/me/ssh_keys` with `{"public_key": "ssh-ed25519 AAAA... user@host"}`. If they have no key pair at all, have them run `ssh-keygen -t ed25519` themselves, then read the `.pub` file. **Never read, print, or send the private key** — it is the file *without* the `.pub` suffix.
3. **Add the remote and push:**

   ```bash
   git config push.autoSetupRemote true
   # deployment_config.ssh_url from the API response, verbatim.
   git remote add miget {deployment_config.ssh_url}
   git push miget
   ```

   A filled-in example, so the shape is unambiguous:

   ```bash
   git remote add miget git@ssh.eu-east-1.migetapp.com:migetmxq/wall-pfhqv.git
   ```

**Option B — HTTPS with a Git token.** Git tokens are not on the API in any form — they cannot be listed, read, or created through it — so this route always ends in the dashboard. Use it only if SSH is unavailable (a network that blocks port 22, or a user who cannot add a key).

1. Send the user to `https://app.miget.com/apps/{APP_UUID}/settings#git_tokens` to reveal or create a token.
   - Default token: the **username** is the resource (miget) name, the **password** is the token value.
   - Any token the user adds: the **username** is the token's name, the **password** is the token value.
   - A token is viewable exactly once. "Token already seen" means it is gone for good and they must create a new one — revoking the default token disables HTTPS deploys entirely.
2. Add the remote and push:

   ```bash
   git config push.autoSetupRemote true
   git remote add miget {deployment_config.https_url}
   git push miget
   ```

   Git prompts for the username and password from step 1. Note the host differs from the SSH one: HTTPS is `git.{region}.miget.io`, SSH is `ssh.{region}.migetapp.com`. The path segments are the same as in the SSH remote.

**For a directory that is not yet a repository**, prefix either option with `git init`, `git add .`, and `git commit -m "initial"`.

Either way, the push triggers a build and deploy — monitor it via `GET /api/v1/apps/{uuid}/deployments` (see "Monitor Deployment" below).

**`public_git`** - Deploy from a public Git repository URL.

```json
{
  "deployment_method": "public_git",
  "deployment_config": {
    "credential_id": "{git-credential-uuid}",
    "repository_url": "https://github.com/user/repo.git",
    "branch": "main",
    "dockerfile_path": "./Dockerfile",
    "build_context": "."
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `repository_url` | string | Yes | - | Full HTTPS Git repository URL (note: `public_git` uses `repository_url`, whereas `github` uses `repository`) |
| `branch` | string | No | `"main"` | Branch to deploy from |
| `credential_id` | string | No | - | UUID of Git credentials (for private repos) |
| `dockerfile_path` | string | No | `"./Dockerfile"` | Path to Dockerfile |
| `build_context` | string | No | `"."` | Docker build context directory |
| `project_path` | string | No | `""` | Subdirectory to build from, for monorepos |
| `run_command` | string | No | - | Override the start command (builder `auto` or `custom`) |
| `language` | string | No | - | Language to build with — **required** when `builder` is `custom` |
| `build_command` | string | No | - | Build command — **required** when `builder` is `custom` |
| `pre_deploy_command` | string | No | - | Command run once before the new release starts (e.g. database migrations) |
| `post_deploy_command` | string | No | - | Command run once after a successful deploy |
| `use_dhi` | boolean | No | `false` | Build on Docker Hardened Images; ignored when `builder` is `dockerfile` |

**Validate the URL before you send it.** The platform enforces the format below and rejects a malformed or unreachable repo at creation — check it yourself first so you can correct the user instead of surfacing a 422:
- Must be an **HTTPS** URL shaped `https://<host>/<owner>/<repo>` (the trailing `.git` is optional — it is normalized server-side). Examples: `https://github.com/rails/rails`, `https://gitlab.com/group/project.git`.
- **Rejected:** SSH URLs (`git@github.com:owner/repo.git`), plain `http://`, a host with a port, and extra path depth such as GitLab subgroups (`host/group/subgroup/repo`) — the check expects exactly host + owner + repo. If the user gives an SSH or browser URL, convert it to the `https://<host>/<owner>/<repo>` form before sending.
- The repo **and** the `branch` must be publicly reachable: on create the platform verifies the repository is accessible and the branch exists. A private repository without a `credential_id`, or a non-existent branch, fails validation. For a private repo, pass a `credential_id` (see Git Credentials).

Three failures come back from that reachability check, and they mean different things — do not treat them all as a bad URL:

| Message | What it means | What to do |
|---|---|---|
| `The repository or branch does not exist.` | The repo is private, the URL is wrong, or the branch name is wrong | Fix the URL or branch, or pass a `credential_id` |
| `The repository host rate limit was exceeded. Please try again later.` | The Git host is throttling the platform — **the URL is fine** | Wait and retry; do not "correct" a URL that was already right |
| `The branch '<name>' has no commits.` | The branch exists but is empty | Push a commit, or point `branch` at one that has history |

**`github`** - Deploy from a GitHub repository using the Miget GitHub App integration.

```json
{
  "deployment_method": "github",
  "deployment_config": {
    "credential_id": "{github-credential-uuid}",
    "repository": "username/repo",
    "branch": "main",
    "auto_deploy_enabled": true,
    "auto_deploy_branch": "main",
    "dockerfile_path": "./Dockerfile",
    "build_context": "."
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `credential_id` | string | Yes | - | UUID of GitHub App credentials (from workspace credentials) |
| `repository` | string | Yes | - | GitHub repository in `owner/repo` format |
| `branch` | string | No | `"main"` | Branch to deploy from |
| `auto_deploy_enabled` | boolean | No | `false` | Automatically deploy when code is pushed |
| `auto_deploy_branch` | string | No | same as `branch` | Branch that triggers auto-deploy |
| `dockerfile_path` | string | No | `"./Dockerfile"` | Path to Dockerfile |
| `build_context` | string | No | `"."` | Docker build context directory |
| `project_path` | string | No | `""` | Subdirectory to build from, for monorepos |
| `run_command` | string | No | - | Override the start command (builder `auto` or `custom`) |
| `language` | string | No | - | Language to build with — **required** when `builder` is `custom` |
| `build_command` | string | No | - | Build command — **required** when `builder` is `custom` |
| `pre_deploy_command` | string | No | - | Command run once before the new release starts (e.g. database migrations) |
| `post_deploy_command` | string | No | - | Command run once after a successful deploy |
| `use_dhi` | boolean | No | `false` | Build on Docker Hardened Images; ignored when `builder` is `dockerfile` |

#### Build Settings for `public_git` and `github`

The two Git-based methods share a set of build fields. They are also accepted on `PUT /api/v1/apps/{uuid}/deployment` under `deployment_config_attributes`, where they behave as a patch — see **Adding a build setting to an existing app** below.

**Adding a build setting to an existing app.** Send the app's current `deployment_method` plus only the fields you want to change; everything you leave out is preserved. To add migrations to a live app you do not have to restate its branch, project path, or commands:

```json
{
  "deployment_method": "public_git",
  "deployment_config_attributes": {
    "repository_url": "https://github.com/user/repo.git",
    "pre_deploy_command": "bin/rails db:migrate"
  }
}
```

To clear a field, send it as an empty string (`"pre_deploy_command": ""`). Note that `repository_url` (for `public_git`) and `repository` + `credential_id` (for `github`) must be present on every request even when unchanged.

**Running database migrations.** Miget has no implicit release phase — nothing runs between the build finishing and the new replicas starting. Put migrations in `pre_deploy_command` so they run **once** before the new release goes live, rather than in the start command where every replica would run them on boot. A Procfile `release` process, or app.json's `scripts.postdeploy` for the post-deploy step, wins over these fields, which apply only when the repository declares none:

```json
{
  "deployment_method": "public_git",
  "deployment_config": {
    "repository_url": "https://github.com/user/repo.git",
    "branch": "main",
    "pre_deploy_command": "npx drizzle-kit migrate"
  }
}
```

Typical values: `npx drizzle-kit migrate` (Drizzle), `alembic upgrade head` (Alembic), `bin/rails db:migrate` (Rails), `python manage.py migrate` (Django). Prisma is the exception — see below. The other Git-based deployment methods have no equivalent field — for those, migrations have to run from the start command or a cronjob.

**What the release phase can and cannot do.** `pre_deploy_command` runs in the runtime image as the unprivileged user `node`, with no package manager and no root. Nothing can be installed from it — `apt-get install …` fails with `Permission denied`. Write the command against what the image already ships.

**Prisma specifically.** The default `auto` runtime image (`node:22.16.0-slim`) has no OpenSSL, which Prisma's migration engine requires. `npx prisma migrate deploy` reaches the database and then fails with `prisma:warn Prisma failed to detect the libssl/openssl version` followed by an empty `Error: Migration engine error:` and `Release failed`. There is no `pre_deploy_command` workaround (see above). For Prisma, use `builder: "dockerfile"` with a base image that includes OpenSSL, or run migrations from outside the platform.

**`NODE_ENV` is `production` during the build.** The generated runtime Dockerfile exports it before `build_command` runs, so a plain `npm ci` skips `devDependencies` — any build needing a bundler or compiler then dies with a module-resolution error such as `Could not find Nx modules`. Use `npm ci --include=dev && npm run build` when the build needs dev dependencies.

**Using `builder: "custom"`.** The `custom` builder needs `language` **and** `build_command` in `deployment_config`; without them the build has nothing to run, and the request is rejected with `422`. `run_command` is optional but usually wanted, since `custom` does not infer a start command:

```json
{
  "builder": "custom",
  "deployment_method": "github",
  "deployment_config": {
    "credential_id": "{github-credential-uuid}",
    "repository": "user/repo",
    "language": "nodejs",
    "build_command": "npm run build",
    "run_command": "node server.js"
  }
}
```

**Monorepos.** Set `project_path` to the subdirectory holding the app (for example `apps/api`). The build then treats that directory as its root.

**`container_registry`** - Deploy a pre-built container image from a registry (Docker Hub, GHCR, etc.). **Paid resources only** — on a free-plan resource every write is rejected with `422`, including a tag change on an app that is already deployed this way.

```json
{
  "deployment_method": "container_registry",
  "deployment_config": {
    "credential_id": "{registry-credential-uuid}",
    "image_url": "docker.io/library/nginx",
    "tag": "latest",
    "command": ["/opt/keycloak/bin/kc.sh"],
    "args": ["start", "--optimized"]
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `image_url` | string | Yes | - | Container image reference **without scheme and without tag** (e.g., `docker.io/library/nginx`) |
| `tag` | string | No | `"latest"` | Container image tag (separate field — do not append it to `image_url`) |
| `credential_id` | string | No | - | UUID of container registry credentials (for private registries) |
| `command` | array of strings | No | - | Override the image's `ENTRYPOINT` (Kubernetes `container.command`). Leave unset to use the image default. |
| `args` | array of strings | No | - | Override the image's `CMD` (Kubernetes `container.args`). Leave unset to use the image default. Use this when an image's ENTRYPOINT is set but no CMD is supplied (e.g. Keycloak prints help on bare run; pass `["start"]` to start the server). |

**Validate the image reference before you send it.** The platform enforces the format below and rejects a malformed reference at creation — check it yourself first:
- Provide `image_url` **without a scheme and without a tag**. Format: `[registry-host[:port]/]namespace/name`, with at least one `/`. Examples: `docker.io/library/nginx`, `ghcr.io/org/app`, `registry.example.com:5000/team/app`.
- **Rejected:** a bare name with no namespace (`nginx` → use `library/nginx`), anything containing `://`, and an image with the tag embedded.
- **Split off the tag.** If the user gives `ghcr.io/org/app:1.2`, send `image_url: "ghcr.io/org/app"` and `tag: "1.2"`. If no tag is given, it defaults to `latest`.
- **Private images** require a matching `credential_id` (a container registry credential — see Container Registry Credentials). Public images (e.g. Docker Hub official images) need none.

**`parent_image`** - Inherit the container image from another app on the platform. When the parent app deploys, this app can auto-sync.

```json
{
  "deployment_method": "parent_image",
  "deployment_config": {
    "parent_app_id": "{parent-app-uuid}",
    "parent_image_auto_sync": true
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `parent_app_id` | string | Yes | - | UUID of the parent application to inherit image from |
| `parent_image_auto_sync` | boolean | No | `false` | Automatically deploy when parent image updates |

**`kamal`** - Deploy from the user's local machine using the Kamal CLI (`kamal deploy`). Miget provides SSH infrastructure and a container registry; the user runs Kamal locally.

```json
{
  "deployment_method": "kamal",
  "deployment_config": {
    "ssh_keys": ["ssh-ed25519 AAAA... user@machine"]
  }
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `ssh_keys` | array of strings | No | `[]` | SSH public keys for Kamal deployment access |

**Kamal-specific requirements:**
- The `builder` should be `"dockerfile"` (Kamal builds images locally using Docker).
- The **registry password** is auto-generated by the platform. After creation, retrieve it via `GET /api/v1/apps/{uuid}` - the response includes the deployment config with the password. The user must set this password as the `MIGET_REGISTRY_PASSWORD` environment variable on their local machine for `kamal deploy` to authenticate with the registry.
- After creation, the app's deployment config will also contain registry hostname, username, image path, and SSH endpoint info needed for the user's `config/deploy.yml`. Retrieve it via `GET /api/v1/apps/{uuid}`.
- The deploy button / `POST /api/v1/apps/{uuid}/deploy` is **not used** for Kamal apps - the user deploys from their machine.
- You **cannot switch** an existing app to Kamal. The app must be created with `deployment_method: "kamal"` from the start.

**Example interaction:**
```
User: "Create an app for me"

Agent: "I'll help you create an application. I need some information first:

Required:
1. Application name (lowercase, alphanumeric with hyphens, e.g., 'my-api-server')
2. Display label (human-readable name, e.g., 'My API Server')
3. Project (which project should this app belong to?)
4. Region (which region should I deploy to?)
5. Builder type (auto/dockerfile/custom)

Optional but recommended:
- Resource (which Miget resource should I assign?)
- Deployment method (git_push/public_git/github/container_registry/parent_image/kamal). Note: the enum value is `container_registry` (not `docker_registry`).

Please provide these details so I can create the application."
```

---

## Walking a customer through their first deploy

The order below is the one that fails least often. Each step names what to
confirm before moving on, because a wrong answer at step 2 is cheap and the same
wrong answer discovered at step 5 costs a rebuild.

1. **Find out what they are deploying.** Ask for the repository, or look at the
   working directory. A `compose.yaml` means a Stack, not an application — stop
   here and read `stacks.md`. A `Dockerfile` means `builder: "dockerfile"`;
   anything else is `builder: "auto"` and the buildpack detects the language.
2. **Pick the resource and confirm the size.** `GET /api/v1/resources`. If the
   workspace has none, creating one costs money, so say the price and wait for a
   yes. Read `assigned` and `project_ids` before offering one — an assigned
   resource refuses projects it is not assigned to.
3. **Create the project** if there isn't a sensible one already.
4. **Create the application**, then read `name` and `public_url` back off the
   response. The name you sent is not the name you got: the platform appends a
   suffix.
5. **Set variables before the first deploy**, not after, so the first boot has
   them. `app_vars_attributes` on create does this in one call.
6. **Deploy, then verify.** `POST /api/v1/apps/{uuid}/deploy`, then poll the
   deployment until it leaves `running`. A `completed` deployment is not the same
   as a working application — check `state` on the app afterwards.

### What to suggest once it is up

Offer these rather than waiting to be asked; each is one call and each prevents a
class of 3am problem:

- **A custom domain**, if they mentioned one — `apps.md` covers the DNS records.
- **Health checks**, so the platform restarts a wedged process instead of serving
  errors. Off by default.
- **Preview environments**, if the app deploys from `github` — a per-pull-request
  copy, configured once.
- **A managed database** instead of one baked into the image, if the app needs
  persistence. See `addons-services.md`.
- **A webhook**, if they will want to know about failed deploys without polling.
  See `webhooks.md`.

## 1. Create and Deploy an Application

All deployment methods follow the same initial steps: create a resource, create a project, then create the app with a `deployment_method` and its corresponding `deployment_config`. The final deploy step varies by method.

```http
# Step 1: Create a resource (if needed)
# Pick plan_code_name from GET /api/v1/plans — never invent one.
POST /api/v1/resources
{
  "plan_code_name": "free",
  "region_code": "eu-east-1"
}

# Step 2: Create a project (if needed)
POST /api/v1/projects
{
  "name": "my-project",
  "description": "My project description"
}

# Step 3: Create the application (example: GitHub deployment)
POST /api/v1/apps
{
  "name": "my-app",
  "label": "My Application",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "auto",
  "ram_size": 256,
  "cpu_size": 0.5,
  "deployment_method": "github",
  "deployment_config": {
    "credential_id": "{github-credential-uuid}",
    "repository": "username/repo",
    "branch": "main",
    "auto_deploy_enabled": true
  }
}

# Step 4: Deploy the application
POST /api/v1/apps/{app-uuid}/deploy
{
  "custom_tag": "v1.2.3",  # Optional: deploy a specific image tag
  "commit_sha": "abc123",  # Optional: deploy specific commit
  "branch": "main"         # Optional: deploy specific branch
}
```

**Alternative: Create with Docker Registry deployment**
```http
POST /api/v1/apps
{
  "name": "my-nginx",
  "label": "My Nginx",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "dockerfile",
  "ram_size": 256,
  "cpu_size": 0.5,
  "deployment_method": "container_registry",
  "deployment_config": {
    "image_url": "docker.io/library/nginx",
    "tag": "latest"
  }
}
```

**Alternative: Create with Kamal deployment**
```http
POST /api/v1/apps
{
  "name": "my-rails-app",
  "label": "My Rails App",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "dockerfile",
  "ram_size": 512,
  "cpu_size": 0.5,
  "deployment_method": "kamal",
  "deployment_config": {
    "ssh_keys": ["ssh-ed25519 AAAA... user@machine"]
  }
}
# Note: Kamal apps are deployed from the user's machine via `kamal deploy`, not via the API
# The registry password is auto-generated - retrieve it via GET /api/v1/apps/{uuid}
```

## 2. Monitor Deployment

```http
# List recent deployments
GET /api/v1/apps/{app-uuid}/deployments?period=7days&status=running

# Get deployment details
GET /api/v1/apps/{app-uuid}/deployments/{deployment-id}

# Stream build logs (SSE)
GET /api/v1/apps/{app-uuid}/deployments/{deployment-id}/stream_logs

# Or get stored logs (after deployment completes)
GET /api/v1/apps/{app-uuid}/deployments/{deployment-id}/logs

# Cancel a running deployment
POST /api/v1/apps/{app-uuid}/deployments/{deployment-id}/cancel

# Rollback to a previous deployment
POST /api/v1/apps/{app-uuid}/deployments/{deployment-id}/rollback
```

## 3. Manage Environment Variables

```http
# List variables
GET /api/v1/apps/{app-uuid}/vars

# Create variable (422 if the key already exists)
POST /api/v1/apps/{app-uuid}/vars
{
  "key": "DATABASE_URL",
  "value": "postgresql://..."
}

# Set variable (identified by key; creates it or overwrites it)
PUT /api/v1/apps/{app-uuid}/vars
{
  "key": "DATABASE_URL",
  "value": "new-value"
}

# Delete variable (identified by key)
DELETE /api/v1/apps/{app-uuid}/vars
{
  "key": "DATABASE_URL"
}
```

## Create App Environment Variable (`POST /api/v1/apps/{uuid}/vars`)

**Required fields:**
- `key` (string) - Variable name (use SCREAMING_SNAKE_CASE, e.g., `DATABASE_URL`)
- `value` (string) - Variable value

**A key the app already has is refused with `422`.** To change a value, or to set one without first checking whether it exists, use `PUT /api/v1/apps/{uuid}/vars` with the same body.

**Ask only what you cannot derive:**
- "What's the variable name? (use SCREAMING_SNAKE_CASE)"
- "What's the variable value?"

## Create App Secret File (`POST /api/v1/apps/{uuid}/secret_files`)

Same body for the project-level endpoint, `POST /api/v1/projects/{project_id}/secret_files`.

**Required fields:**
- `filename` (string) - Path the file is mounted at inside the container, e.g. `/app/config/service-account.json`. Unique within the app
- `text` (string) - File contents. Encrypted at rest and never returned by any endpoint

**Ask only what you cannot derive:** you usually know both from the file the user pointed at — read the path and the contents rather than asking. Confirm the in-container path when it differs from where the file sits locally.

**Read back the `name`** from the response and store it; it is the only handle for updating or deleting the file later, and you cannot re-derive it from `filename`.

## 5. Add Custom Domain

```http
# Add domain
POST /api/v1/apps/{app-uuid}/domains
{
  "domain": "api.example.com"
}

# Update domain (e.g., enable SSL)
PUT /api/v1/apps/{app-uuid}/domains/{domain-uuid}
{
  "ssl_enabled": true
}
```

## Create App Domain (`POST /api/v1/apps/{uuid}/domains`)

**Required fields:**
- `domain.name` (string) - Fully qualified domain name (e.g., `"app.example.com"`)

**Ask only what you cannot derive:**
- "What domain name should I add? (e.g., app.example.com)"

## 9. Manage App Ports

App ports are managed through the standard app-UUID-scoped endpoints. Port `5000` is auto-created for HTTP traffic; use these endpoints to add extra TCP/UDP ports.

```http
# List all ports
GET /api/v1/apps/{app-uuid}/ports

# Create a new port
POST /api/v1/apps/{app-uuid}/ports
{
  "internal_port": 8080,
  "protocol": "tcp",
  "public": false
}

# Get port details
GET /api/v1/apps/{app-uuid}/ports/{port-id}

# Expose port publicly
PATCH /api/v1/apps/{app-uuid}/ports/{port-id}/expose_publicly

# Make port private
PATCH /api/v1/apps/{app-uuid}/ports/{port-id}/make_private

# Delete port
DELETE /api/v1/apps/{app-uuid}/ports/{port-id}
```

## Create App Port (`POST /api/v1/apps/{uuid}/ports`)

**Required fields:**
- `internal_port` (integer) - Internal port number (1-65535)
- `protocol` (string) - Protocol: `"tcp"` or `"udp"`

**Optional but important:**
- `public` (boolean) - Whether the port should be publicly accessible (default: `false`)

**Important notes:**
- Requires `apps:operate` permission
- Port management is not available on free plan resources
- Ports can be exposed publicly or made private after creation using separate endpoints
- Port `5000` is fixed for HTTP traffic on the app's `*.migetapp.com` URL — it is auto-created, cannot be removed or changed, and the app must listen on it. Use this endpoint to add extra TCP/UDP ports for custom protocols; they are **private by default** — use `expose_publicly` to make them reachable from outside the cluster. See https://docs.miget.com/networking/ports for the full list of supported ports.
- **Read the address back, do not construct it.** A public port answers with `public_endpoint` — `hostname:port`, the thing to hand the user or put in a connection string — plus `external_port` and the `endpoint_hostname` / `endpoint_ipv4_primary` / `endpoint_ipv6_primary` fields for the proxy it landed on. The external port is assigned from a pool, so it is never the internal one, and a private port returns `null` for all of them.

**Ask only what you cannot derive:**
- "What internal port number? (1-65535)"
- "What protocol? (tcp or udp)"
- "Should this port be publicly accessible? (true/false)"

## Create App Cronjob (`POST /api/v1/apps/{uuid}/cronjobs`)

**Required fields:**
- `label` (string) - Human-readable display name
- `command` (string) - Shell command to execute
- `cron` (string) - Cron expression (e.g., `"0 * * * *"` for hourly). Required only when `schedule_type` is `"cron"`

**Optional but important:**
- `schedule_type` (string) - `"cron"` for custom cron expression, `"interval"` for predefined intervals (default `"interval"`)
- `interval_type` (string) - `"every_10_minutes"`, `"hourly"`, or `"daily"` (for interval type)
- `daily_time` (string) - Execution time for daily jobs in HH:MM format (24-hour)
- `minute` (string) - Minute component for scheduling (0-59)
- `hour` (string) - Hour component for scheduling (0-23)
- `size` (string) - A size code from `GET /api/v1/apps/{uuid}/cronjobs/sizes`, e.g. `hobby-1c-1g`: the CPU and RAM each run gets. The codes are the resource plans' (`hobby-…` shared CPU, `pro-…` dedicated), and the app's CPU class decides which it is offered, whether the app runs on a resource or on Usage Billing. A minute is priced at the monthly price of the resource plan of that size over 43,200 minutes (`cents_per_minute`); on a resource, runs draw on the cron minutes its plan includes first, a run at a bigger size drawing more of them. Defaults to the smallest size with at least the app's RAM, whatever the app's CPU (the largest if none has that much); a size of the other class is refused with `422`
- `max_run_seconds` (integer) - How long a run may take, from 1 to 86400 seconds (24 hours). A run still going then is stopped and ends `failed` with reason `DeadlineExceeded`. Leave it out or send `null` for no limit
- `overage` (string) - What it does once its resource's cron minutes for the billing cycle are used up: `pause` (the default) until the next renewal, or `bill`, each run past them billed by the second at its size's price per minute on Usage Billing. **`bill` needs `apps:manage`** (403 otherwise; `pause` needs only `apps:operate`) and starts the $5 monthly minimum when the subscription pays for nothing else (422 with `checkout_url`). It applies only where the resource has cron minutes and is not on an enterprise contract; elsewhere it is ignored and the response's `overage` says what happens: `bill` on Usage Billing and on a plan that costs nothing, which include none, `pause` on a contract with a cron minute limit, `null` where no run is billed

**Important notes:**
- The identifier the platform runs the job under is generated, not chosen — it comes back as `name` (`cronjob-lhs2i`). A `name` you send is ignored.
- Updating a cronjob (`PUT`) only changes `label`, `command`, `size`, `max_run_seconds` and `overage`; a new size resizes the runs that follow, `max_run_seconds: null` removes the limit (leaving it out keeps it), and switching `overage` to `bill` resumes it if the used-up cron minutes paused it. The response carries it as `size`; `null` is a cron job created before sizes, which keeps its CPU and RAM until one is set.
- `suspended: true` in the response is a paused cronjob. Besides `POST .../pause`, the platform pauses cron jobs when the workspace is suspended, when a resource is stopped for non-payment, when the spend limit is reached or a usage charge is refused, and when their resource's cron minutes are used up and they are set to pause; the last three resume them by themselves, once the limit is raised, the charge paid or the next renewal starts the minutes again. Resume the others with `POST .../resume`. A cron job created set to pause while its resource's minutes are used up starts paused.
- **Cron jobs are billed for the time their runs take, at their size, on any resource plan including the free one — on a resource only past the cron minutes its plan includes** (1,000 a cycle at the plan's own size; `cron_minutes` on `GET /api/v1/resources/{uuid}`), and only for a cron job whose `overage` is `bill`; otherwise it pauses there until the renewal. On a resource exempted from billing that is the only charge it can bring, on an enterprise contract there is none, and on an app on Usage Billing or a resource on a plan that costs nothing every run is billed. The first cron job that is billed in a workspace whose subscription pays for nothing is refused with `422 {"errors": "...", "checkout_url": "..."}` for the $5 monthly minimum; send the person to `checkout_url`, then repeat the request. On a workspace in its free trial, the trial carries on and covers the plan only: runs are charged to the card while it runs, the first time once usage reaches $5, then at the charge threshold (see `references/workspace.md`). Usage never ends the trial. The **schedule cannot be changed in place** — to reschedule, DELETE the cronjob and create a new one.
- Per-run logs are available via `GET /api/v1/apps/{uuid}/cronjobs/{id}/stream_logs` (SSE) once the job has run at least once.
- Whether runs worked: `GET .../cronjobs/{id}/runs`. A failed run also opens a `cronjob_failed` issue on the cron job (`GET /api/v1/issues?app_id={uuid}`), which its next successful run resolves; the cron job keeps its schedule either way. Runs from before the platform recorded them are not listed.

**Ask only what you cannot derive:**
- "What display label should I use?"
- "What schedule type? (cron for custom expression, interval for predefined)"
- "If interval, which interval? (every_10_minutes/hourly/daily)"
- "If cron, what cron expression? (e.g., 0 * * * * for hourly)"
- "If daily, what time? (HH:MM format)"
- "What command should be executed?"

## Set Preview Environment Config (`PUT /api/v1/apps/{uuid}/preview_environments/config`)

No field is required — the first PUT creates the config, filling anything you omit with the defaults below.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Create preview environments for this app |
| `resource_mode` | string | `parent` | `parent` runs them where the app runs; `existing` requires `resource_id`; `pay_per_use` (`workspace_pool` on an enterprise contract) runs them without a resource, in the app's region. An app without a resource takes only `parent` |
| `tier` | string | the app's | For preview environments without a resource: `"hobby"` for shared CPU, `"pro"` for dedicated; implied by `size`. Cloned add-ons run on the same class |
| `size` | string | the app's | For preview environments without a resource: a size code from `GET /api/v1/workspace/pay_per_use/sizes`, e.g. `hobby-1c-2g`. `""` goes back to the app's size; a size the class does not offer is a `422` |
| `resource_id` | string | — | Resource **UUID** preview environments run on. Required when `resource_mode` is `existing` |
| `project_id` | string | — | Project **UUID** they land in. Omit to follow the parent app |
| `trigger_mode` | string | `pull_request` | `pull_request` (one per PR) or `every_branch` (one per pushed branch) |
| `trigger_filter` | string | `auto` | Which PRs qualify: `auto` (all non-draft), `label`, `branch_pattern`, `pr_name_pattern` |
| `trigger_label` | string | — | PR label that triggers creation. Required when `trigger_filter` is `label` |
| `branch_pattern` | string | — | Glob against the branch name, e.g. `feature/*`. Required when `trigger_filter` is `branch_pattern` |
| `pr_name_pattern` | string | — | Glob against the PR title, e.g. `feat:*`. Required when `trigger_filter` is `pr_name_pattern` |
| `auto_deploy_on_push` | boolean | `true` | Redeploy on every push to the tracked branch |
| `cleanup_on_pr_merge` | boolean | `true` | Destroy when the PR is merged |
| `cleanup_on_pr_close` | boolean | `false` | Destroy when the PR is closed unmerged |
| `cleanup_on_branch_delete` | boolean | `false` | Destroy when the branch is deleted |
| `cleanup_on_inactivity` | boolean | `false` | Destroy after `inactivity_days` without a commit |
| `inactivity_days` | integer | `14` | Days without a commit before cleanup |
| `retention_days` | integer | `7` | Days kept before expiry |
| `retention_mode` | string | `after_pr_close` | Count retention from PR close, or `from_creation` |
| `comment_on_github` | boolean | `true` | Post the environment URL as a PR comment. Forced off in `every_branch` mode |
| `clone_settings` | object | `{}` | What is copied from the parent app |

**`clone_settings` reads as "everything unless told otherwise".** `clone_variables`, `clone_secret_files` and `clone_security` are treated as `true` when absent, so you only ever set them to `false`. An absent or empty `addons`/`cronjobs` key means *clone them all*; supplying either narrows to what you list.

A pattern or label the filter requires but that you leave empty is a **422**, so send `trigger_filter` and its companion field together.

## Create Container Registry Credential (`POST /api/v1/container_registry_credentials`)

Stores credentials for pulling images from a private registry. The returned `uuid` is passed as `deployment_config.credential_id` when creating an app with `deployment_method: container_registry` (and is also usable for `github`/`public_git` private repositories). The `token` is encrypted at rest and never returned.

**Required fields:**
- `name` (string) - Display name (must be unique within the workspace)
- `registry` (string) - Provider: `docker_hub`, `github`, `gitlab`, `aws_ecr`, `azure`, `digitalocean`, `quay`, or `generic`
- `username` (string) - Registry username
- `token` (string) - Registry password or access token

**Optional:**
- `registry_hostname` (string) - Registry hostname URL (required for `generic`, `aws_ecr`, and `azure`; optional otherwise)
- `skip_validation` (boolean) - Skip live credential validation (non-production environments only)

**Response fields:** `uuid`, `name`, `registry`, `username`, `registry_hostname`, `created_at`, `updated_at` (the `token` is never returned).

**Ask only what you cannot derive:**
- "Which registry provider? (docker_hub, github, gitlab, aws_ecr, azure, digitalocean, quay, generic)"
- "What's the registry username and access token/password?"
- "For generic/aws_ecr/azure registries: what's the registry hostname?"

## Rollback Deployment (`POST /api/v1/apps/{uuid}/deployments/{id}/rollback`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `id` (string, path) - Deployment UUID

**Constraints:**
- Deployment must be rollbackable (has image URL, same deployment method as app, not running/failed)
- Application must be in deployable state
- Requires `apps:deploy` permission

**Ask only what you cannot derive:**
- "Which deployment should I rollback to? (provide deployment UUID)"

## Update Security Settings (`PUT /api/v1/apps/{uuid}/security`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- At least one of the following optional fields must be provided

**Optional fields:**
- `allow_connections` (boolean) - Allow other applications on the same resource (miget) to connect to this app over the internal network
- `basic_auth_enabled` (boolean) - Enable Basic Authentication for the application
- `basic_auth_username` (string) - Username for Basic Authentication (required when `basic_auth_enabled` is true)
- `basic_auth_password` (string) - Password for Basic Authentication (required when `basic_auth_enabled` is true, leave blank to keep current password)

**Constraints:**
- Requires `apps:deploy`, `apps:operate` or `apps:manage`
- When `basic_auth_enabled` is true, both `basic_auth_username` and `basic_auth_password` are required (unless password already exists and you want to keep it)
- The app response returns `basic_auth_enabled` so you can tell whether Basic Auth is enforced, but Basic Auth **credentials are never returned** by the API.

**Ask only what you cannot derive:**
- "Should I enable Basic Authentication? (true/false)"
- "If enabling Basic Auth, what username should I use?"
- "If enabling Basic Auth, what password should I use? (leave blank to keep current)"
- "Should I allow internal network connections? (true/false)"

## Update Health Checks (`PUT /api/v1/apps/{uuid}/health_checks`)

Configures Kubernetes health probes (liveness, readiness, startup).

**Required fields:**
- `uuid` (string, path) - Application UUID

**Optional fields (all optional, provide at least one):**
- `liveness_probe_enabled` (boolean) - Enable liveness probe
- `readiness_probe_enabled` (boolean) - Enable readiness probe
- `startup_probe_enabled` (boolean) - Enable startup probe
- `liveness_probe_path` (string) - Liveness probe HTTP path
- `readiness_probe_path` (string) - Readiness probe HTTP path
- `startup_probe_path` (string) - Startup probe HTTP path
- `*_probe_initial_delay_seconds` (integer) - Seconds before first probe check
- `*_probe_timeout_seconds` (integer) - Seconds before probe times out
- `*_probe_period_seconds` (integer) - Seconds between probe checks
- `*_probe_failure_threshold` (integer) - Consecutive failures before marking unhealthy
- `*_in_app_failure_notification_enabled` (boolean) - In-app notifications on probe failure
- `*_email_failure_notification_enabled` (boolean) - Email notifications on probe failure

Replace `*` with `liveness`, `readiness`, or `startup`.

## Update Scaling Profile (`PUT /api/v1/apps/{uuid}/scaling_profile`)

Configures auto-scaling. Not available on free plan.

**Required fields:**
- `uuid` (string, path) - Application UUID

**Optional fields:**
- `replicas` (integer) - Fixed number of running instances
- `auto_scaling_enabled` (boolean) - Enable automatic horizontal scaling
- `auto_min_replicas` (integer) - Minimum instances when auto-scaling
- `auto_max_replicas` (integer) - Maximum instances when auto-scaling
- `cpu_threshold` (integer) - CPU usage % that triggers scale-up (1-100)
- `memory_threshold` (integer) - Memory usage % that triggers scale-up (1-100)
- `period_enabled` (boolean) - Enable time-based scaling windows
- `scaling_start_time` (string) - Start time for scaling window (HH:MM, 24-hour)
- `scaling_end_time` (string) - End time for scaling window (HH:MM, 24-hour)
- `within_resources` (boolean, default `true`) - `false` scales an app on a resource **beyond its plan**: see below.

**Scaling beyond the plan.** With `within_resources: false`, the first replica of an app on a resource runs on that resource as before, and every replica past it, up to the workspace's `replicas_per_process` limit (10 on Personal, 20 on Team; `GET /api/v1/workspace/limits`) in `replicas`, or `auto_max_replicas` when autoscaling, is billed by usage while it runs:
- on a resource with **shared CPU**, for its RAM only — the extra replicas share the resource's CPU;
- on a resource with **dedicated CPU**, for its CPU and RAM at the dedicated rate — the resource's CPU is raised by the extra replicas.

Turning it on takes `apps:manage`; turning it back off, and changing replicas afterwards, takes what any scaling change takes. The resource keeps only the first replica booked, so it has room for other workloads. It is refused with `422` on a resource nobody pays for (such as the free plan), an app without a resource, an enterprise contract, an app with RWO storage, or while Usage Billing is turned off for the workspace. The first time it may need the $5 Usage Billing minimum (see Usage Billing in `references/workspace.md`): the request is then refused with `422 {"errors": "...", "checkout_url": "..."}` and nothing changes until a person completes that checkout. `within_resources: true` brings every replica back onto the resource and is refused with `422` when they do not fit — send fewer `replicas` in the same request. The workspace spend limit and the app's monthly budget both take such an app back to the one replica its resource holds rather than stopping it; while the spend limit holds it, more than one replica is refused with `422`. Its extra replicas are listed with every other workload in `GET /api/v1/workspace/usage`.
- `max_monthly_budget` (integer or null) - **Only for an app billed for its replicas**: on Usage Billing, or on a resource with `within_resources: false`. A budget in whole US dollars for this application's own usage in a billing cycle; its add-ons are not counted. `null` removes it. Sent for any other app, it is refused with `422`.
- `budget_notification_enabled` (boolean) - Email the workspace admins at 50% and 75% of the budget and when it is reached, each once per billing cycle.
- `stop_at_budget_limit_enabled` (boolean) - At the budget, scale the application down to one replica.

**Reaching the budget** is checked every minute. With `stop_at_budget_limit_enabled`, the application is scaled down to one replica with auto-scaling off. It is not stopped: it keeps serving, and its storage and that one replica keep being billed. While it is held there, a request that would run more than one replica is refused with `422`; other settings still save. It gets back exactly the scaling it had:

- at the start of the next billing cycle, or once it runs again if it was stopped then;
- at once, when the budget is raised above what the application has spent this cycle, removed, or `stop_at_budget_limit_enabled` is turned off. Replica settings sent in that same request are kept instead.

Without `stop_at_budget_limit_enabled`, reaching the budget only sends the email. A changed budget does not send the warnings it is already past.

Whether an app is held there right now is `held_at_monthly_budget` in `GET /api/v1/apps/{uuid}` (and in the app lists). Check it before scaling up an app on Usage Billing, and tell the user why a scale-up would be refused instead of retrying.

## Scale a Worker (`PUT /api/v1/apps/{uuid}/workers/{name}`)

Scales one worker, a process the Procfile runs besides the web, without touching the web or the app's other workers. Takes `apps:deploy`, `apps:operate` or `apps:manage`, as the app's own scaling does. Not available on free plan (`403`).

**Required fields:**
- `uuid` (string, path) - Application UUID
- `name` (string, path) - The worker's `name` as `GET /api/v1/apps/{uuid}` lists it under `workers` (`404` for a name the app does not have)

**Optional fields** (the app's own scaling fields, without the budget or `within_resources`):
- `replicas` (integer) - Fixed number of replicas
- `auto_scaling_enabled` (boolean) - Scale between `auto_min_replicas` and `auto_max_replicas` on the worker's CPU or memory use
- `auto_min_replicas` (integer) / `auto_max_replicas` (integer) - The range when auto-scaling; the minimum is at least `1`, since autoscaling a worker to none is refused with `422`
- `cpu_threshold` (integer) / `memory_threshold` (integer) - Usage % that triggers scale-up (1-100)
- `period_enabled` (boolean) - `false` clears the daily window
- `scaling_start_time` / `scaling_end_time` (string, HH:MM, 24-hour, UTC) - A daily window in which it runs `auto_max_replicas`

**How many it may run.** Growing past this answers `422` with the reason in `errors`; scaling down is always accepted:
- on Usage Billing or in the workspace pool, the workspace's `replicas_per_process` limit (`GET /api/v1/workspace/limits`; `POST /api/v1/workspace/limits/increase_requests` asks for more); in the pool, also only what the pool has room for;
- on a resource, as many as fit in the RAM left after everything else on it, the workers recorded before this one included;
- on free compute, none until the workspace starts Usage Billing: free compute covers the web alone;
- whatever the above allows, a worker whose app.json size has no match in the app's CPU class is capped to at most one replica; naming a matching size in app.json's `formation` lifts it.

**It lasts until the next deploy.** As for anything app.json declares, the next deploy sets the replicas app.json's `formation` gives this worker again; a worker the `formation` does not name keeps the count you set. To keep a count, put it in app.json.

Returns the worker (`name`, `state`, `replicas`, `quota`); `replicas` is the new count at once (the minimum while it auto-scales), and `state` follows once the cluster reports it. A worker of an app on Usage Billing is billed at its own size for the replicas it runs, from when they run. The app's monthly budget counts its workers' usage, but reaching it scales only the web down.

## Change Application State (`PATCH /api/v1/apps/{uuid}/state`)

**Required fields:**
- `uuid` (string, path) - Application UUID
- `state` (string) - Target state: `schedule_start`, `schedule_stop`, or `schedule_restart`

Note: apps use `schedule_*` values. Addons and services use `process_*` values (see below) — they are not interchangeable.

## Clone Application (`POST /api/v1/apps/{uuid}/clone`)

Creates a **new application** from an existing one. It is not a snapshot and not a
backup: it is a fresh app that starts out configured like the source.

**Every copy flag defaults to `false`.** A clone with no flags gets the source app's
own settings (ports, resource limits, and the like) and nothing else — no environment
variables, no secret files, no add-ons, no cronjobs. Ask the user what they want
carried over rather than guessing, and pass the flags explicitly.

#### The one decision that changes everything: where the code comes from

**The source app's deployment configuration is not copied.** A clone therefore has no
source to build from, and its `deployment_method` falls back to `git_push`. You have
two ways to finish the job, and you must pick one:

- **`use_parent_image: true`** — the clone runs the *same container image* as the
  source. This is the fast path: nothing to build, nothing to configure, and adding
  `parent_image_auto_sync: true` redeploys the clone whenever the source's image
  changes. Use it for extra environments or extra regions of the same code.
- **Leave it `false`** — the clone is an independent app, and you must give it a
  source afterwards with `PUT /api/v1/apps/{uuid}/deployment` (see that endpoint;
  changing `deployment_method` rebuilds the config, so send every field it needs).
  Use it when the clone will diverge from the source.

If you skip this decision, the user ends up with an app that cannot deploy.

#### Collect the UUIDs first

`addons` and `cronjobs` take UUIDs from the **source** app, so read them before you
clone — you cannot discover them afterwards:

```bash
curl -H "Authorization: Bearer $MIGET_API_TOKEN" https://app.miget.com/api/v1/apps/{source-uuid}/addons
curl -H "Authorization: Bearer $MIGET_API_TOKEN" https://app.miget.com/api/v1/apps/{source-uuid}/cronjobs
```

`clone_data` on an add-on copies its **contents** — database rows, stored files. Without
it you get an empty add-on of the same type. Confirm this one with the user explicitly:
copying a production database into a new app is rarely what someone means by "clone my
app", and it is not reversible from here.

**Required fields:**
- `uuid` (string, path) - Source application UUID to clone from
- `label` (string) - Display name for the cloned application
- `name` (string) - Unique service name for the cloned application (lowercase, alphanumeric with hyphens)
- `project_id` (string) - UUID of the project the clone is created in
- `resource_id` (string) - UUID of the compute resource (Miget) the clone runs on. **Not required when `billing` is `"pay_per_use"` or `"workspace_pool"`** — see below. Otherwise, leaving it out is refused with a `422`.

**Optional fields:**
- `use_parent_image` (boolean, default: false) - Run the source app's image instead of building
- `parent_image_auto_sync` (boolean, default: false) - Redeploy the clone when the parent's image changes. Only meaningful with `use_parent_image`
- `clone_variables` (boolean, default: false) - Copy environment variables
- `clone_secret_files` (boolean, default: false) - Copy secret files
- `clone_scaling_settings` (boolean, default: false) - Copy the scaling profile (replicas, autoscaling)
- `clone_health_checks` (boolean, default: false) - Copy liveness/readiness/startup probe config
- `clone_security` (boolean, default: false) - Copy security settings (allowed connections and Basic Auth)
- `addons` (array, default: []) - Add-ons to clone, each `{uuid: String, clone_data: Boolean}`
- `cronjobs` (array, default: []) - Cronjob UUIDs to clone

```json
{
  "label": "Acme API (staging)",
  "name": "acme-api-staging",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "use_parent_image": true,
  "parent_image_auto_sync": true,
  "clone_variables": true,
  "clone_secret_files": true,
  "clone_security": true,
  "addons": [{"uuid": "{addon-uuid}", "clone_data": false}],
  "cronjobs": []
}
```

#### Cloning onto Usage Billing or the workspace pool

A clone does not have to run where its source does. Instead of `resource_id`, send the same placement a new app takes (see "Usage Billing" under Create Application):

- `billing` (string) - `"pay_per_use"`, or on an enterprise contract `"workspace_pool"`; `resource_id` is then ignored. An unknown value is refused with `400`.
- `region_code` (string) - Where the clone runs. **Defaults to the source app's region**, not to any region the workspace runs in. `clone_data` on an add-on is refused with `422` when the clone runs in another region than its source.
- `tier` (string) - `"hobby"` (a shared slice of CPU) or `"pro"` (dedicated cores), implied by `size` when one is given. **Defaults to the source app's CPU class** (Pro when the source runs on a dedicated-CPU resource or on Usage Billing with dedicated CPU). Permanent for the clone.
- `size` (string) - A size code from `GET /api/v1/workspace/pay_per_use/sizes`. **Leaving it out keeps the source app's CPU and RAM**, with CPU capped at the largest standard plan of the CPU class. `cpu_size`, `ram_size` and the free size are refused with `422` — a clone is never free compute — and so is `size` on a clone onto a resource.

The application and every add-on you clone are each held to what one workload without a resource can have: more RAM or disk than that is refused with `422` before the clone is created, and an add-on's CPU is capped there. A clone that is the first workload without a resource in its region and CPU class is refused with `422` while that capacity is set up — repeat the request after a minute. If the workspace has to pay the $5 monthly minimum first, the request is refused with `422 {"error": "...", "checkout_url": "..."}` and no app is created; send the person to `checkout_url`, then repeat the request. The response carries `pay_per_use`, `workspace_pool` and `miget: null` as a new app placed that way does.

#### After the call

The clone comes back in state `cloning` and is provisioned in the background. Poll
`GET /api/v1/apps/{uuid}` until the state settles, then:

- If you did **not** use `use_parent_image`, configure its deployment source now — the
  app cannot deploy until you do.
- Environment variables often name the source app (hostnames, callback URLs, database
  names). Read them back with `GET /api/v1/apps/{uuid}/vars` and tell the user which
  ones look like they need changing. Do not silently rewrite them.
- The clone gets its own URL, derived from its `name`. Hand it over the same way you
  would for a new app.

## 10. Create and Deploy with Kamal

Kamal is a deployment method where the user deploys from their local machine using `kamal deploy`. Unlike other methods, Miget does not build or deploy the app - it provides the infrastructure (SSH endpoint, container registry) and the user runs Kamal locally.

**Important constraints:**
- Apps must be created with Kamal from the start - you cannot switch an existing app to Kamal
- The deploy button is not used for Kamal apps (the user deploys from their machine)
- The registry password is auto-generated by the platform - retrieve it via `GET /api/v1/apps/{uuid}`

```http
# Step 1: Create resource and project (same as other methods)

# Step 2: Create app with Kamal deployment method
POST /api/v1/apps
{
  "name": "my-rails-app",
  "label": "My Rails App",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "dockerfile",
  "deployment_method": "kamal",
  "deployment_config": {
    "ssh_keys": ["ssh-ed25519 AAAA... user@machine"]
  },
  "ram_size": 512,
  "cpu_size": 0.5
}

# Step 3: Get app details to retrieve deploy.yml values
GET /api/v1/apps/{app-uuid}
# Response includes deployment_config with all values needed for config/deploy.yml:
# {
#   "deployment_method": "kamal",
#   "deployment_config": {
#     "type": "kamal",
#     "ssh_keys": ["ssh-ed25519 AAAA... user@machine"],
#     "registry_image": "registry.eu-east-1.miget.io/my-resource/my-rails-app",
#     "registry_hostname": "registry.eu-east-1.miget.io",
#     "registry_username": "my-resource",
#     "registry_password": "auto-generated-password"
#   }
# }

# Step 4: Update SSH keys (if needed)
PUT /api/v1/apps/{app-uuid}/deployment
{
  "deployment_method": "kamal",
  "deployment_config_attributes": {
    "ssh_keys": ["ssh-ed25519 AAAA... user@machine", "ssh-rsa AAAA... other@machine"]
  }
}

# Step 5: User deploys from their machine using Kamal CLI
# kamal deploy (from the user's local machine, not via API)
```

## Example: Complete Application Setup (GitHub)

```http
# 1. Authenticate — every request below carries this header
Authorization: Bearer $MIGET_API_TOKEN

# 2. Get or create resource
GET /api/v1/resources
# If none exists (plan_code_name comes from GET /api/v1/plans):
POST /api/v1/resources
{
  "plan_code_name": "free",
  "region_code": "eu-east-1"
}

# 3. Create project
POST /api/v1/projects
{
  "name": "my-api",
  "description": "REST API project"
}

# 4. Create application with GitHub deployment
POST /api/v1/apps
{
  "name": "api-server",
  "label": "API Server",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "auto",
  "ram_size": 256,
  "cpu_size": 0.5,
  "deployment_method": "github",
  "deployment_config": {
    "credential_id": "{github-credential-uuid}",
    "repository": "username/api-repo",
    "branch": "main",
    "auto_deploy_enabled": true
  }
}

# 5. Add environment variables
POST /api/v1/apps/{app-uuid}/vars
{
  "key": "NODE_ENV",
  "value": "production"
}

# 6. Add database addon (label and postgres_version are required)
POST /api/v1/apps/{app-uuid}/addons
{
  "type": "postgres",
  "label": "Primary database",
  "postgres_version": "17"
}

# 7. Deploy (optional: specify commit_sha, branch, or custom_tag)
POST /api/v1/apps/{app-uuid}/deploy
{
  "commit_sha": "abc123",  # Optional: deploy specific commit (for Git/GitHub)
  "branch": "main"         # Optional: deploy specific branch (for GitHub)
}

# 8. Monitor deployment
GET /api/v1/apps/{app-uuid}/deployments?period=7days&status=running
# Wait for status: "completed"

# 8b. Rollback if needed (to a previous deployment)
POST /api/v1/apps/{app-uuid}/deployments/{previous-deployment-id}/rollback

# 9. Add custom domain
POST /api/v1/apps/{app-uuid}/domains
{
  "domain": "api.example.com"
}
```

## Example: Complete Kamal Application Setup

```http
# 1. Authenticate (same as above)

# 2. Get or create resource (same as above)

# 3. Create project (same as above)

# 4. Create application with Kamal deployment
POST /api/v1/apps
{
  "name": "my-rails-app",
  "label": "My Rails App",
  "project_id": "{project-uuid}",
  "resource_id": "{resource-uuid}",
  "builder": "dockerfile",
  "ram_size": 512,
  "cpu_size": 0.5,
  "deployment_method": "kamal",
  "deployment_config": {
    "ssh_keys": ["ssh-ed25519 AAAA... user@machine"]
  }
}

# 5. Retrieve app details for deploy.yml configuration
GET /api/v1/apps/{app-uuid}
# Response includes deployment_config with:
#   - registry hostname, username, image path, and registry password
#   - SSH endpoint hostname and port
# Use these values to create your local config/deploy.yml
# Set the registry password as MIGET_REGISTRY_PASSWORD env var locally

# 6. (Optional) Add environment variables for the app
POST /api/v1/apps/{app-uuid}/vars
{"key": "RAILS_ENV", "value": "production"}

POST /api/v1/apps/{app-uuid}/vars
{"key": "SECRET_KEY_BASE", "value": "your-secret-key"}

# 7. User deploys from their local machine:
#    kamal setup   (first time)
#    kamal deploy  (subsequent deploys)
# These commands are run locally, NOT via the API
```

---
