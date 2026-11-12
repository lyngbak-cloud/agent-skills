# Stacks (Docker Compose)

A stack deploys a multi-service application from one `docker-compose.yml` in a Git
repository. Miget reads the compose file and provisions the underlying
applications and managed services for you.

Read this the moment a repository turns out to have a compose file. Creating
applications one by one from a compose repo produces something that looks right
and is not a stack — it will not reconcile on the next deploy, and the services
will not be wired together.

## Contents

- [What a stack is](#stacks-docker-compose)
- [Endpoints](#endpoints)
- [Creating one](#create-stack-post-apiv1stacks)
- [From the Miget catalogue](#11a-deploy-a-known-app-from-the-miget-catalogue-deployablesh)
- [From a plain compose file](#11b-repos-with-a-compose-file-but-no-composemigetyaml)

## Stacks (Docker Compose)

A **Stack** deploys a multi-service application from a single `docker-compose.yml` in a Git repository.

- A stack is pinned to a **Resource** (Miget) and belongs to a **Project**
- Miget detects the compose services and provisions the underlying **apps** and **managed services** for you
- The stack tracks a Git **branch**; each deploy re-reads the compose file and reconciles changes
- Stack `state` and per-service status are computed from the underlying apps/services

---

## Endpoints

## Stacks (Docker Compose)

Stacks reuse `apps:*` permissions (read = `apps:view`, create/delete = `apps:manage`, deploy/config = `apps:deploy`/`operate`/`manage`).

- `GET /api/v1/stacks` - List all stacks. Each entry also carries `compose_path` and `deployment_config`, the same as the detail response
- `POST /api/v1/stacks/analyze` - Detect compose services and required env vars from a repo (creates nothing; call before creating)
- `POST /api/v1/stacks` - Create a stack (analyzes the repo server-side, then provisions the apps/services)
- `GET /api/v1/stacks/{uuid}` - Get stack details (computed `state`, `services`, `latest_deployment`, `deployment_config`). `deployment_config` holds `type` (`github` for a stack with a credential, `public_git` without), `repository` (owner/repo slug for `github`, `null` for `public_git`), `repository_url`, `branch`, `auto_deploy_enabled` (`null` for `public_git`) and `credential_id`, the UUID of the stored Git credential, `null` when none is set; `credential_id` and `repository_url` are the fields `POST /stacks` took. `repository_url` is normalized by the platform, so it can differ in form from the text you sent (a `.git` suffix, the host spelling)
- `PUT /api/v1/stacks/{uuid}` - Update a stack (`label`, `compose_path`). `project_id` moves the stack, and every application and service it runs, to another project (requires `apps:manage`)
- `DELETE /api/v1/stacks/{uuid}` - Delete a stack (cascades to its apps and services)
- `POST /api/v1/stacks/{uuid}/deploy` - Trigger a redeploy (optional: `commit_sha`)
- `PUT /api/v1/stacks/{uuid}/deployment` - Update the GitHub deployment config (`branch`, `auto_deploy_enabled`, `repository`)
- `GET /api/v1/stacks/{uuid}/deployments` - List deployment history
- `GET /api/v1/stacks/{uuid}/deployments/{id}` - Get a single deployment (`{id}` is the deployment UUID)

## Create Stack (`POST /api/v1/stacks`)

A stack deploys a multi-service app from a `docker-compose.yml` in a Git repository. The compose source is analyzed server-side, so **call `POST /api/v1/stacks/analyze` first** to discover the services and which environment variables are required.

**Required fields:**
- `repository_url` (string) - Git repository URL (GitHub or public HTTPS Git)
- `branch` (string) - Git branch to deploy
- `resource_id` (string) - UUID of the compute resource (Miget) to deploy onto
- One of `project_id` (string, existing project UUID) **or** `new_project_name` (string, creates a new project)

**Optional:**
- `compose_path` (string) - Path to the compose file in the repo (default `"."`)
- `credential_id` (string) - UUID of a stored Git credential (for private repositories)
- `label` (string) - Display name; `name` (string) - codename seed (derived from the repo if omitted)
- `new_project_description` (string) - description when creating a new project
- `env_var_overrides` (object) - Values for required env vars, shaped `{ "<service>": { "<KEY>": "<value>" } }`
- `auto_populate_required_vars` (boolean) - Fill any required env var left without a custom value (default `false`)

**Discover-then-supply flow (handling env vars):**
1. `POST /api/v1/stacks/analyze` with `{repository_url, branch, compose_path?}`. Each app/standalone service in the returned `manifest` has `env_vars: [{ key, value, required }]`.
2. Required vars are those with `required: true` and a blank `value`. Ask the user whether each should be a **custom** value or **auto-populated** (good for secrets).
3. `POST /api/v1/stacks` with custom values in `env_var_overrides` and/or `auto_populate_required_vars: true`. Managed services (databases/caches) are auto-configured — never supply env vars for them.

**Derived variables:** some stacks declare a variable that must be computed from another rather than
chosen freely — for example a Supabase `ANON_KEY` is a JWT signed with that stack's `JWT_SECRET`. The
platform always computes those from their source, so a value you send for one in `env_var_overrides`
is ignored. Supply the source variable (or let `auto_populate_required_vars` generate it) and the
dependent values are derived to match.

**Variables on redeploy:** `POST /api/v1/stacks/{uuid}/deploy` keeps every value saved for the stack.
A required variable added to the compose file since the last deploy is generated only when the compose
file gives it a generation rule (`generate`/`length`), and derived variables are computed from their
source. A new required variable with no such rule is left unset; the redeploy does not fail on it.

**Error responses to handle:**
- `422 { "error": "Missing required environment variables: web.SECRET, ..." }` - a required var was neither supplied nor auto-populated; go back to step 2.
- `422 { "error": "Not enough capacity on the resource: ..." }` - the manifest needs more RAM/disk than the resource has; pick a larger `resource_id`.

---

## 11. Create a Compose Stack (Docker Compose)

```http
# Step 1: Analyze the repo to discover services and required env vars (creates nothing)
POST /api/v1/stacks/analyze
{
  "repository_url": "https://github.com/acme/shop.git",
  "branch": "main",
  "compose_path": "."
}
# Response: { "manifest": { "apps": [...], "managed_services": [...] }, "warnings": [] }
# Inspect each service's env_vars for entries with "required": true and a blank "value",
# then ask the user whether to supply them or auto-populate (good for secrets).

# Step 2: Create the stack, supplying required env vars
POST /api/v1/stacks
{
  "repository_url": "https://github.com/acme/shop.git",
  "branch": "main",
  "resource_id": "{miget-uuid}",
  "project_id": "{project-uuid}",
  "label": "Shop",
  "env_var_overrides": { "web": { "STRIPE_API_KEY": "sk_live_..." } },
  "auto_populate_required_vars": true
}

# Step 3: Watch deployment progress
GET /api/v1/stacks/{stack-uuid}/deployments
```

## 11a. Deploy a Known App from the Miget Catalogue (deployable.sh)

Miget curates ready-to-run, platform-tuned Compose stacks — WordPress, Ghost, n8n, Kafka,
Supabase, Metabase, and many more — in the public **deployable.sh** catalogue (repo
`deployable-sh/stacks`). Each stack directory ships a `compose.miget.yaml` carrying the
platform overrides a raw compose file lacks (port 5000 ingress, `private: true` defaults,
RAM sizing, managed-service wiring), so these deploy correctly out of the box.

**When the user asks to run a well-known self-hostable app (e.g. "deploy WordPress", "spin up
Ghost"), prefer the catalogue over an arbitrary compose file found on the web** — a random
`docker-compose.yml` from the internet almost never has Miget's overrides and will likely fail
to deploy.

1. **Find it in the catalogue.** Look the app up in `deployable-sh/stacks`: list the repo's
   top-level directories (or browse https://deployable.sh) and match the app to a directory. The
   slug is the app's lowercased name (e.g. `wordpress`, `ghost`, `n8n`). Do **not** search the
   internet for a compose file when the app exists here.
2. **Deploy it** as a normal Compose Stack (section 11), pointing `repository_url` at the
   catalogue repo and `compose_path` at the stack directory:

```http
POST /api/v1/stacks/analyze
{
  "repository_url": "https://github.com/deployable-sh/stacks.git",
  "branch": "main",
  "compose_path": "wordpress"
}
# Then POST /api/v1/stacks with the same source plus any required env vars (Step 2 above).
```

**If the app is not in the catalogue**, don't grab a random compose file off the web — ask the
user for a repository (public Git or GitHub) instead.

## 11b. Repos with a compose file but no `compose.miget.yaml`

When deploying a user's **own** repository whose base `docker-compose.yml` has no
`compose.miget.yaml` beside it, the stack still deploys, but without Miget's per-service tuning —
services fall back to default sizing and exposure. **Offer to create a `compose.miget.yaml`
overlay** (a sibling of the compose file) and, once the user confirms, generate it and add it to
their repo. Miget merges it onto the base compose at detect/deploy time; it carries only
`x-miget` overrides:

```yaml
# compose.miget.yaml — Miget overlay, merged onto your docker-compose at deploy time
services:
  web:                 # the public HTTP entry — must listen on port 5000 (Miget's only ingress port)
    x-miget:
      ram: "1024"      # memory: plain MB, or a unit like "1Gi"
  worker:
    x-miget:
      ram: "512"
      private: true    # internal only, not publicly exposed
      vpc: default       # join this workspace VPC (its "default" subnet), or { name: default, subnet: backend }
      project_variables: true  # inherit the project's environment variables (paid plans)
  db:                  # a database/cache -> provision as a managed add-on, not a raw container
    x-miget:
      managed: postgres  # supported: postgres, valkey
      cpu: "500m"
      ram: "1Gi"
      storage: "5Gi"
  cache:
    x-miget:
      managed: valkey
      ram: "256Mi"
volumes:
  webdata:
    x-miget: { size: "5000", type: RWO }   # disk: MB or a unit; RWO (default) or RWX (shared)
```

Rules when generating it:
- Give every service an `x-miget.ram` (plain MB or a unit like `1Gi`); add `cpu` (e.g. `500m`) when known.
- Exactly one service is the public HTTP entry and must listen on **port 5000**; mark every other
  service `x-miget: { private: true }`.
- For databases/caches, use `x-miget.managed: <postgres|valkey>` (with `storage`) so Miget runs them
  as managed add-ons and injects their connection variables — don't run them as raw containers.
- Give every named volume an `x-miget: { size: "<MB or unit>", type: RWO }` (`RWX` only when the
  volume is shared across replicas).
- A named volume mounted by more than one app or standalone service becomes one shared storage
  service of that size, billed once, and each of them mounts it at its own path and sees the same
  files. It is listed among the stack's items with `compose_service_type: volume` and
  `compose_service_name: volume:<name>`. A volume only one service mounts stays that service's own
  storage.
- A volume keeps the kind it was first deployed as. A shared volume stays shared even when only one
  service mounts it later. A volume first deployed as one service's own storage never becomes
  shared: a deploy that adds a second service mounting it fails with
  ``Volume `<name>` is block storage of `<service>` and cannot be shared with `<service>`.``
- When no service mounts a shared volume any more, it is dropped from the stack's items and its
  storage service is kept as an orphaned service with its data. It still counts against the
  resource's disk until it is deleted.
- `x-miget.vpc` names a VPC of the workspace in the resource's region, by name, or as
  `{ name, subnet }` for a subnet other than `default`. The service is created on it, and a
  redeploy attaches it when the key is added and detaches it when the key is removed; moving an
  attached service to another subnet is not applied (remove the key, deploy, then add the new one). A VPC name that does not exist
  fails that service's creation.
- `x-miget.project_variables: true` makes the service inherit the project's environment
  variables (the same switch as `PUT /api/v1/apps/{uuid}/vars/project_variables_enabled`);
  removing it or setting `false` turns inheritance off on the next deploy. Keep values that
  differ per service out of the project, since an app variable with the same key wins.

Re-run `POST /api/v1/stacks/analyze` after adding the file to confirm the detected services and
sizing. Full field reference: the "Docker Compose Stacks" page in the Miget docs.

---
