# Workspace, resources, projects and people

The container everything else sits in: the workspace and its bill, the compute
resources that cost money, the projects that partition them, and the permissions
that decide what a token may do.

Read this before creating a resource — that is the call that spends money — and
whenever a `403` needs explaining or a project turns out to be invisible rather
than forbidden.

## Contents

- [Workspaces and the bill](#workspaces)
- [Usage Billing](#usage-billing)
- [Workspace pool](#workspace-pool)
- [Resources](#resources-migets)
- [Projects](#projects)
- [Endpoints](#endpoints)
- [Permissions](#permissions--authorization)
- [Users and SSH keys](#users)

## Workspaces

Miget uses **workspace-based multi-tenancy**. Each user can belong to multiple workspaces (organizations/teams).

- **Default workspace:** If no `X-Workspace-Id` header is provided, API uses the user's default workspace
- **Multi-workspace:** Include `X-Workspace-Id` header to specify which workspace to operate on:
  ```http
  X-Workspace-Id: {workspace-uuid}
  ```

## Stopping the bill

**None of this is on the API, and you must not try to approximate it by deleting things.** When a user wants to stop paying, downgrade, leave, or wind a workspace down, your job is to tell them exactly where to click and what will happen — then stop.

| What the user wants | Where they go | What it does |
|---|---|---|
| Stop paying entirely | **Billing** in the sidebar → **Cancel Subscription** on the subscription card | Charges stop now. Everything keeps running until the end of the paid period, then the workspace is suspended and everything in it stopped. Five days later the resources and all their applications, services, buckets and databases are **permanently deleted**. Resumable from the same page until the period ends, and restorable from it during those five days. |
| Stop paying, **enterprise workspace** | Their account manager, or support@miget.com | There is no cancel button on an enterprise plan and no endpoint behind it — notice period, final invoice and data handover follow the signed agreement. Billing shows an "Ending your contract" note instead. Do not tell an enterprise user to look for a Cancel button; they will not find one. |
| Drop one resource, keep the rest | **Resources** → pick it → **Settings** → **Delete** | Deletes that resource and everything on it, and takes it off the bill. The subscription continues for the workspace plan and any other resources. |
| Change plan | **Settings** → **Plan** | |
| Remove the workspace | **Settings** → **Delete** | Owner only — the tab is not shown to anyone else. Revokes the subscription immediately with no refund for the remaining period, and deletes every project and application. The last remaining workspace cannot be deleted. |
| Payment method or invoices | **Billing** → **Update Payment Methods** | Opens the payment provider's portal. |

Two things to warn about, because a user will not expect either:

- **Deleting applications does not reduce the bill.** Billing is per resource — the capacity applications run on — not per application. An empty resource costs exactly what a full one does. This is the single most common billing surprise, so say it before the user starts deleting apps to save money.
- **Cancelling ends in deletion, not in a frozen account.** A cancelled workspace is suspended and its workloads stopped, and five days later the resources are destroyed. Those five days are the whole window: **Billing** → **Restore Subscription** keeps everything, and after them nothing can be recovered. Restoring reopens the workspace with its resources intact; applications are not started for the user, so tell them to start what they want once it is back. Tell them to restore or export within the window.
- **Restore first, then change the plan.** While the subscription has ended, `POST /api/v1/resources` and `PUT /api/v1/resources/{uuid}` refuse anything but the free plan with `422` and `Your subscription has ended. Restore it in Billing first, then try again.` There is no endpoint that restores, so send the user to **Billing** → **Restore Subscription**, and make the change once it is paid.

**A resource that costs nothing is not touched by any of this.** A free-plan resource — and one an admin has taken off the bill — keeps running while the paid ones are stopped, and survives the deletion at the end of the window. Say so when a user asks what they lose by not paying: they keep the free tier, which is also where the workspace lands afterwards.

A workspace can also be **suspended** by the platform — after a cancellation completes, for non-payment, when its usage reaches the spend limit, when a usage charge taken at its threshold is refused, or when an application is blocked for abuse. It is a consequence, never something a user or an agent triggers. Only the spend limit one has an endpoint that lifts it: raising or removing the limit. A refused usage charge lifts itself once the charge is paid — it is retried every hour, and a person can retry it from the Usage page in the dashboard after updating the card.

What a suspension blocks depends on why it happened:

| Reason | What still works over the API | The `403` message |
|---|---|---|
| Unpaid subscription, still being billed | `GET` and `DELETE` — read anything, and delete anything, to cut the bill. The dashboard follows the same rule and adds the plan change | `Your workspace is suspended because the subscription is not paid. Settle it in Billing and everything comes back. Reading and deleting resources stay available in the meantime.` |
| Subscription ended, inside the five-day window | `GET` only — nothing is being charged, and the resources are what a restore brings back | `Your subscription has ended, so this workspace is read-only until it is restored in Billing. Nothing has been deleted yet.` |
| Usage reached the spend limit | `GET` and `DELETE`, and `PUT /api/v1/workspace/spend_limit`, which starts the stopped workloads on Usage Billing again once the limit is above the cycle's usage | `This workspace reached its spend limit, so it is read-only and its workloads on Usage Billing are stopped. Raise the limit with PUT /api/v1/workspace/spend_limit and they start again. Reading and deleting stay available.` |
| A usage charge was refused | `GET` and `DELETE`; the stopped workloads on Usage Billing start again once the charge is paid | `The last usage charge for this workspace was refused, so it is read-only and its workloads on Usage Billing are stopped. The charge is retried every hour; once it is paid they start again. Update the payment method in Billing, or retry the payment from the Usage page. Reading and deleting stay available.` |
| Abuse (an application was blocked) | nothing | `Your workspace has been suspended due to suspicious activity in one of your applications. Please contact support to reactivate it.` |
| Anything else | nothing | `Your workspace has been suspended. Please contact support to reactivate it.` |

`POST /api/v1/buckets/{uuid}/objects/download_url` counts as a read: it keeps working under every hold except abuse, so files can still be pulled out of a workspace that is read-only.

Do not report any of these as a permission problem the user can fix by changing a role, and do not retry the blocked call. Relay which one it was, since they need different people: billing for the first, a workspace admin raising the limit for the spend limit, whoever holds the card for a refused usage charge, support for the others. On the unpaid one, the reads that still work are worth using — you can show the user what they are about to lose before they decide.

## Resources (Migets)

A **Resource** (internally called "Miget") is a compute resource that provides CPU, RAM, and disk space. Resources are assigned to applications and services.

- Resources have **plans** (free, dev, pro tiers)
- Resources can have **components** (extra RAM, CPU, disk)
- Resources can have **labels** (user-defined strings like "production", "staging", "sandbox" for identification)
- Resources are region-specific
- Resource capacity is reported as `total_ram_size` / `total_used_ram_size` / `available_ram_size` (and the `disk_size` equivalents) — all in **bytes**, the same unit as `quota.ram_size` on apps and `ram_size` on plans. `*_cpu_size` fields are fractional core counts. When you check whether a resource can host another app, compare bytes to bytes; treating these as MiB is the usual cause of "it should have fit".
- A resource is a fixed-capacity compute unit that hosts as many workloads as fit. It is **not** one-per-app, and by default it is not owned by any project
- A resource can be **assigned** to one or more projects (`POST /api/v1/projects/{project_id}/resources`). Once assigned, only those projects may place workloads on it; anything else is refused with **422**. Assigning restricts the **resource**, never the project — a project can always use any unassigned resource, whether or not it has assignments of its own. `GET /api/v1/projects/{project_id}` lists a project's assigned resources under `resources`
- Every resource reports its own assignments as `assigned` (a boolean) plus `project_ids` — an array of project UUIDs. `assigned: false` means nobody has assigned it and any project may deploy on it. `project_ids` lists the projects it is assigned to that **you can access**; a resource assigned solely to projects you cannot access is not returned to you at all, so you will not see it in `GET /api/v1/resources` and cannot address it by UUID
- **Resource selection can be refused.** Before offering a `resource_id`, read `assigned` and `project_ids`: the resource is usable when `assigned` is false, or when `project_ids` contains the target project's UUID. Prefer a resource assigned to the target project, then an unassigned one. A 422 mentioning an assignment means the resource is closed to that project — pick another one, or assign it to the project first
- **Assigning is itself refused while the resource still runs workloads the assignment would lock out.** The 422 names the blocking projects with their workload counts — except that it **redacts every project you cannot reach**, reporting only how many there are. Send `with_hosting_projects: true` on `POST /api/v1/projects/{project_id}/resources` to assign the resource to those projects too and let the call through; it only ever widens access, and every workload already there stays where it is. When a blocker is redacted this flag is the *only* way forward, because you cannot name a project you cannot see — do not retry the bare call and do not report the assignment as impossible

## Projects

**Projects** are logical groupings of apps and services.

- A project holds applications, static sites, services, stacks and buckets. Applications, static sites, services and stacks always belong to exactly one project; buckets may belong to one
- Ownership (which project a workload belongs to) and placement (which resource it runs on) are separate. Static sites sit on no resource at all
- **Ownership can be changed on every kind; placement cannot.** Send `project_id` to `PUT /apps/{uuid}`, `PUT /services/{id}`, `PUT /stacks/{uuid}`, `PUT /static/{uuid}` or `PUT /buckets/{uuid}` to move a workload between projects — it keeps running on the same resource throughout. Moving a stack moves every application and service in it. There is no way to move a workload to a different resource; that requires deleting and recreating it
- Because the resource cannot change, a move is refused with **422** when the resource is assigned to projects that do not include the destination. The way through is to assign the resource to the destination project as well, not to pick a different resource
- A `project_id` that does not exist — or that exists but you cannot reach — is a **404** carrying `{"error": "Project not found"}`, on every one of those endpoints. Those two cases answer identically on purpose: a status code that told them apart would confirm that a restricted project is there. Read this 404 as "the destination could not be resolved", never as "the workload is gone" — the workload is untouched and still in its original project. Re-read `GET /api/v1/projects` to see which destinations you may actually use
- Projects can have **project-level environment variables** and **project secret files**, shared across the apps in the project. Neither is automatic: an app receives them only while its own `project_variables_enabled` / `project_files_enabled` toggle is on. Both are managed over the API — see "Project Environment Variables" and "Project Secret Files"
- Projects can have **assigned resources**, returned as `resources` on the project response. An empty list means the project simply uses the shared pool — which it may do even when it has assignments. The same relationship reads from the other side as `assigned` and `project_ids` on the resource response, which is the cheaper check when you are choosing a resource
- A project can also be **restricted** to specific people, returned as `restrictions` on the project response. An empty list means the project is open to the whole workspace; any entry closes it to everyone but the listed members, everyone holding a listed role, the workspace owner and holders of the **built-in `admin` role**. A custom role never bypasses a restriction, whatever permissions it carries — including `workspace:members`. Restricting projects is available on organization and enterprise workspaces only, and that plan refusal is checked **before** the subject you passed, so on a smaller plan you get the plan message even when the email is also wrong
- **A project you cannot reach simply does not appear.** It is absent from `GET /api/v1/projects`, and so are its applications, services, stacks and buckets in their own listings; addressing any of them by UUID returns **404**, not 403. So a short project list is not proof that the workspace holds nothing else — it is what you are allowed to see. The same is true of resources assigned solely to projects you cannot reach; an unassigned resource stays visible to everyone, because any project may deploy on it.

## Usage Billing

Every application, service, bucket and stack is created one of two ways, and the choice cannot be changed afterwards:

| Placement | What it means |
|---|---|
| On a resource | Capacity bought as a plan (`resource_id`), shared by everything placed on it for one monthly price. |
| Usage Billing | No resource. Created with `billing: "pay_per_use"`, a `region_code` and a `size`, and billed for what it reserves while it runs. |

What is charged, per unit for a full 720-hour month: RAM $5.00/GiB, shared CPU $8.00/core, dedicated CPU $24.00/core, storage $0.15/GiB. CPU and RAM are what the workload reserves (its size, on every replica), charged for the seconds it runs, **with no minimum per run**; storage is charged around the clock, because a stopped workload keeps its volume. A build is charged only while it replaces pods that were already running: the first build of an app that is not running costs nothing, and a build you cancel reads `cancelled`, not failed. A bucket is the exception: it is charged $0.02/GiB a month for the data it holds, measured every hour, and its `disk_size` is only the most it may hold; its line in the usage breakdown is `object_storage`, not `disk`. A cron job is billed for the seconds its runs take at its size's price per minute, on one `run` line; `GET /api/v1/apps/{uuid}/cronjobs/sizes` lists each size's price per minute, which is its resource plan's monthly price over 43,200 minutes. **On a resource, the runs draw on the cron minutes its plan includes first** (see "Cron minutes" under Resources): what those cover is billed nothing and counts `0` in the usage breakdown, and past them each cron job pauses or is billed, as its `overage` says. A cron job never takes RAM from its resource: it runs as a pod of its own, on any resource, and only its cron minutes limit it. On an enterprise contract its runs cost nothing, and are unlimited unless the contract sets cron minutes; on a resource exempted from billing they cost something only past its cron minutes, for a cron job set to `bill`. The largest workload on Usage Billing is the largest standard plan of its CPU class, and a volume or bucket at most 1 TB; each instance of a PostgreSQL cluster is held to that on its own, and is billed on its own.

**Free compute.** A personal workspace that pays for nothing yet can run free workloads on Usage Billing, created with `billing: "pay_per_use"` and `size: "free"` — `GET /api/v1/workspace/pay_per_use/sizes?workload=app|postgres|valkey` lists the free size first while the account may take it. Free compute belongs to the account (the workspace owner), at no charge, starts no minimum, and needs a card on file:

| Workload | Size | How many per account |
|---|---|---|
| App (`POST /api/v1/apps`) | 0.1 CPU, 512 MiB RAM | Any number, sharing **750 running hours a calendar month** |
| PostgreSQL service (`POST /api/v1/services`, `service_type: "postgres"`) | 0.1 CPU, 256 MiB RAM, 1 GiB disk, one instance | One |
| Valkey service (`POST /api/v1/services`, `service_type: "valkey"`) | 0.1 CPU, 32 MiB RAM, 0.1 GiB disk | One |

MySQL has no free size. **A free app sleeps after 15 minutes without traffic** and wakes on the next request, answered with a `503` while it starts — retry with a backoff before reporting a failure. Only running time counts against the hours: an asleep or stopped app counts nothing. **When the hours run out, every free app of the account is stopped until the 1st of next month**, when they start again. Until then a free app does not start — `PATCH /api/v1/apps/{uuid}/state` with `schedule_start` still answers that the start is scheduled, but the app stays `stopped` — and creating another free app is refused with a `422` saying so. Give an app a paid size to run it before the 1st. The owner is emailed at 50% and 75% of the hours and when they are used. The free services are outside the hours. **A free PostgreSQL sleeps when no client is connected** and wakes on the next connection; asleep for 30 days it enters a 14-day grace period and on day 44 it is permanently deleted, with emails to the workspace admins on days 23 and 29 and at each step. A free Valkey does not sleep. A second free service of a type is refused with a `422` until the first is deleted or given a paid size. Free workloads keep the free plan's limits: one replica, no backups, no public ports, no container registry deploys. **A free workload cannot join a private network**: attaching one to a VPC, or creating a free app with `vpc_uuid` or `vpc_subnet_uuid`, is refused with a `422` until it has a paid size. **Free compute covers an app's web process alone**: its workers (other processes in the Procfile or app.json `formation`) are held at no replicas, each with an `awaiting_checkout` issue (`GET /api/v1/issues`), until the workspace starts Usage Billing — `POST /api/v1/workspace/pay_per_use/start` starts the minimum without creating anything, and each held worker then starts at one replica (the next deploy brings back app.json's quantity). An add-on on a free app (`POST /api/v1/apps/{uuid}/addons`) is billed like any workload on Usage Billing, so it starts the minimum first, with the `checkout_url` refusal below when the subscription pays for nothing. `size: "free"` is the only way to ask for free compute: sending `free_tier` to any other endpoint changes nothing. Asking a free workload for any other size afterwards starts Usage Billing, with the `checkout_url` refusal below when the subscription pays for nothing; the workload then leaves the free hours and stops sleeping, unless the same request sends `auto_sleep: true`. Free compute and the free plan exclude each other: an account that took free compute cannot create a free resource, and one holding a free resource is not offered free compute.

**The first workload on Usage Billing starts a $5 monthly minimum**, charged once per workspace however many workloads are on Usage Billing, and counted towards their usage: a cycle's usage is billed only beyond it. **The workspace's credits pay the usage beyond the minimum before the card does**: its credit balance (`GET /api/v1/workspace/credits`) pays it as it is billed, whatever other discount the subscription carries, and only what the credits leave is charged, at the threshold or with the renewal (`credits_applied_in_cents` in the usage response). A workspace billed by enterprise agreement never spends them. **A subscription that already pays for anything else — a resource, seats, an add-on — is charged no minimum**, and all of its usage is billed; `minimum_in_cents` in the usage response is then `0`. While a payment is overdue, creating anything with `billing: "pay_per_use"` is refused with a `422`. **On a workspace in its free trial, Usage Billing keeps the trial, which covers the plan only**: usage is paid and charged to the card while the trial runs, the first time once it reaches $5 (`charge_threshold_in_cents` is `500` until a charge in the trial is paid, or lower where Miget set a lower value), then at the threshold below; what is left when the trial ends is charged right after it when it came to $5 or more (less is billed with the first renewal). The trial itself runs to its end — tell the user before creating it. It is added to the subscription when that workload is created, prorated for the cycle, and ends after a renewal once nothing is on Usage Billing any more. A workspace with no subscription to add it to is answered with a `422` and a `checkout_url`; nothing is created until a person completes that checkout, after which the same request succeeds. Usage is charged with the renewal that closes the cycle, also in a cycle in which something else was bought — **and before it, whenever the usage not yet charged reaches the workspace's threshold** (`charge_threshold_in_cents` in the usage response). The threshold starts at $25 and every paid renewal raises it a step ($50, $250, $1,000, $4,000); after that usage is charged with the renewal alone. It can also be raised on request (see "Limits"); a value Miget set for the workspace replaces the ladder. What was charged early is not billed again: the renewal bills only the usage beyond it (`charged_early_in_cents`). **If the card refuses such a charge, every workload on Usage Billing is stopped** and every cron job billed by the run is paused — as at the spend limit — until it is paid; see the suspension table above. Deleting a workspace that still owes usage ends its subscription at the end of the period, where that last usage is charged, rather than at once. **Before a workspace's first payment** — while Miget runs it free during an onboarding — Usage Billing is still metered, a calendar month at a time, but nothing is charged and there is no minimum or threshold: right after the first payment all of it is charged at once (from $5; less is billed with the first renewal), the workspace's credits paying first, as `due_after_first_payment_in_cents` in the usage response says until then. Deleting the workspace before that payment leaves it on record as owed.

A workload on Usage Billing comes back with `billing: "pay_per_use"` and no resource (`miget` is `null`). It never appears under `GET /api/v1/resources`.

`GET /api/v1/workspace/usage` reports what the cycle has cost so far, the minimum, a projection for the cycle if what runs now keeps running and the day that pace reaches the spend limit, and what each workload contributed so far, the hours not yet closed included. It needs the `workspace:billing` permission.

A workspace admin can cap the usage bill with `PUT /api/v1/workspace/spend_limit`. **At the limit every application, database and service on Usage Billing is stopped** — nothing on a resource — an app on a resource scaling beyond its plan goes back to the one replica its resource holds, and every cron job billed by the run is paused, on a resource too. Raising the limit starts back exactly the ones that were stopped, resumes those cron jobs and gives those apps their replicas back. The limit cannot be set below `spend_limit_floor_in_cents` from the usage response — the monthly minimum, or what storage alone costs for a cycle when that is more; before the subscription's first payment there is no minimum, so storage alone, and free workloads count nothing — and sending `null` removes it.

Usage Billing is **on by default** and a workspace admin can turn it off with `PUT /api/v1/workspace/pay_per_use` (`{"enabled": false}`), which is refused with `422` while anything is still on Usage Billing. While it is off, `billing: "pay_per_use"` is refused with `422` on every create endpoint; `pay_per_use_enabled` in the usage response says which it is.

## Workspace pool

On an **enterprise contract** (a workspace with an agreed CPU, RAM and storage capacity) Usage Billing is not offered; instead an application, service, bucket or stack can be placed straight in the **workspace pool** with `billing: "workspace_pool"`, a `region_code` and a `size` (or `tier` where there is CPU and no size). Like Usage Billing it has no resource and the choice is permanent, but nothing is billed by the hour: it draws on the contract's capacity.

| Placement | What it means |
|---|---|
| On a resource | A resource inside the contract, counted against the pool at its full size. |
| Workspace pool | No resource. Counted against the pool by what it reserves: CPU and RAM on every replica (at the minimum replica count when autoscaled), storage once per volume or database instance, whether it is running or not. |

**A full pool refuses, with `422`**: creating a workload, growing its CPU, RAM or storage, adding replicas or database instances, deploying a stack, or buying a resource that the remaining capacity does not cover. The message names what ran out — `The workspace pool has no room left for this (RAM). Free some capacity or ask to raise the pool.` Shrinking always works. Only Miget changes the pool; tell the user to contact their account manager.

`billing: "workspace_pool"` is refused with `422` on a workspace without a contract, and an unknown `billing` with `400`. A workload in the pool comes back with `billing: "workspace_pool"` and no resource (`miget` is `null`). `GET /api/v1/workspace/usage` is refused with `422` on a contract — there is no usage bill.

## Limits

`GET /api/v1/workspace/limits` lists every limit the workspace runs into, so read
it before an action that could cross one rather than finding out from a `422`.
Any member may read it. Each entry has a `key`, its `value` in its `unit`
(`replicas`, `mounts`, `databases`, `buckets`, `cents`, `minutes`, `cores` or `bytes`; `null` means there is none),
`usage` (the most the workspace uses of it now, or `null`), `adjustable` and
`increase_requested`.

| `key` | What it limits | Raised on request |
|---|---|---|
| `replicas_per_process` | Replicas, or the autoscaling maximum, of one process (the web or a worker) on Usage Billing or scaling beyond its resource's plan; 20 on Team. On a resource, the resource's capacity decides | Yes |
| `mounts_per_volume` | Applications mounting one shared storage; 10 on Team | Yes |
| `databases_per_workspace` | PostgreSQL, MySQL and Valkey databases in the workspace, read replicas included, on an app or as a service; 40 on Team. A database the build creates from app.json is never refused | Yes |
| `buckets_per_workspace` | Buckets in the workspace; 100 on Team | Yes |
| `charge_threshold` | Unbilled usage at which usage is charged before the renewal (see above). Not listed on an enterprise contract | Yes, while there is one |
| `cron_minutes_per_resource` | Cron minutes a resource's plan includes each billing cycle, 1,000 at the plan's own size; `usage` is the most any resource has used this cycle. On an enterprise contract `null` (unlimited) unless Miget set a limit, which you cannot request | No |
| `hobby_process_cpu`, `hobby_process_ram`, `pro_process_cpu`, `pro_process_ram` | The largest single workload of each CPU class on Usage Billing or in the workspace pool | No — a larger one needs a resource |
| `volume_size` | The largest disk one such workload holds | No |

The first four follow the workspace plan: Personal has half of Team's (10 replicas,
5 mounts, 20 databases, 50 buckets), Organization one and a half times (30, 15, 60,
150) and Enterprise twice (40, 20, 80, 200). A limit Miget set for the workspace, on
request or not, replaces its plan's value and stays when the workspace changes
plan, so moving to a higher plan does not raise it.

A refusal for crossing a raisable limit ends with "Ask for more on the Limits
page." To ask, `POST /api/v1/workspace/limits/increase_requests` with `limit`,
`requested_value` in the limit's unit (cents for `charge_threshold`) above its
current value, and a `reason`. It needs `workspace:billing`, which a workspace API
token cannot carry, so it takes a personal token of a member who holds it. Miget
reviews it by hand: the requester is emailed when it is approved — possibly at a
different value than asked — or declined with a reason. One request per limit waits
at a time; a second is refused with `422`. Tell the user it is not immediate.

## Endpoints

## Workspace Billing

- `GET /api/v1/workspace/usage` - This cycle's usage — `billed_in_cents` (the closed hours plus adjustments), `pending_in_cents` (every hour not closed yet, from `billed_through` to now, priced live), `total_in_cents` (the two together), `billed_through` (the start of the first hour not closed yet) and `cycle_start`/`cycle_end` — the monthly minimum (`minimum_in_cents`), the projection for the cycle if what runs now keeps running (`projected_in_cents`, never below the minimum) and the day that pace reaches the spend limit (`limit_reached_on`, or null), the spend limit, the threshold usage is charged at before the renewal (`charge_threshold_in_cents`, or null once it is charged with the renewal alone, or where Miget waived it for the workspace) and what was charged at it this cycle (`charged_early_in_cents`), what the workspace's credits pay of the cycle's usage beyond the minimum, the hours not yet closed included (`credits_applied_in_cents`), what ran before the subscription's first payment and is charged right after it (`due_after_first_payment_in_cents`; the threshold is null until then and the cycle is the calendar month), a per-workload breakdown and `adjustments` — corrections Miget made by hand, each with a signed `amount_in_cents` (negative is a credit) and a `note`. Adjustments count toward the total the moment they are made. Requires `workspace:billing`
- `PUT /api/v1/workspace/spend_limit` - Set or clear the ceiling on usage (`spend_limit_in_cents`, required: the ceiling, or `null` for none; a body without it is a `400`). Requires `workspace:billing`. A value below `spend_limit_floor_in_cents` is refused with a `422`
- `GET /api/v1/workspace/pay_per_use/sizes` - The sizes an app, service or add-on on Usage Billing or in the workspace pool can take, both CPU classes unless `tier` (`"hobby"` shared, `"pro"` dedicated) is given. `workload` (`app`, the default, `postgres` or `valkey`) picks whose free size is listed first, while the account may take it (see "Free compute"). Each has an `id` to send as `size` — its size code, e.g. `hobby-1c-2g`, which is also the code of the resource plan of that CPU and RAM where there is one (`hobby-0.25c-512m`, `hobby-0.5c-512m`, `hobby-0.5c-1g` and `hobby-1c-512m` have none), and it implies the tier — `cpu`, `ram_mb`, `tier`, `free` (true for the free size), and `hourly_in_cents` / `monthly_in_cents` for one replica or instance running (`null` on an enterprise contract). Any member can read it
- `GET /api/v1/workspace/limits` - Every limit the workspace runs into, with its value, what is in use and whether it can be raised (see "Limits"). Any member can read it
- `GET /api/v1/workspace/credits` - The workspace's credits (`balance_in_cents`), what of them expires next (`expiring_in_cents`, `expires_at`) and the 50 newest operations. A workspace's credits come from members who transfer theirs and from Miget (a program's, `granted`, possibly with an expiry date: those are spent first, and what is left of them on that day expires), go on its purchases and renewals, and pay for Usage Billing beyond the monthly minimum. Requires `workspace:billing`
- `POST /api/v1/workspace/credits/transfer` - Move the authenticated user's whole credit balance into the workspace, for good: a transfer cannot be undone, and the credits stay with the workspace whoever owns it later. Requires `workspace:billing`, which a workspace API token cannot carry, so it takes a personal token. `422` when there is nothing to transfer, the workspace is on an enterprise plan, or it is suspended for review. Answers the workspace's credits as the `GET` does
- `POST /api/v1/workspace/limits/increase_requests` - Ask for a limit to be raised (`limit`, `requested_value`, `reason`); returns the request with `status: "pending"`. Requires `workspace:billing`
- `POST /api/v1/workspace/pay_per_use/start` - Start Usage Billing without creating anything — what a free-compute app's workers wait for. Answers the usage when it is already on Usage Billing or has just started it; a workspace with no subscription to add the minimum to gets `422` with a `checkout_url` a person completes. Refused with `422` while Usage Billing is off, a payment is overdue, or on an enterprise contract. Requires `workspace:billing`
- `PUT /api/v1/workspace/pay_per_use` - Turn Usage Billing on or off (`enabled`). Requires `workspace:billing`. Turning it off while anything is on Usage Billing (an app with `within_resources: false` included), or on an enterprise contract, is refused with a `422`. Turning it off also turns off every preview environment config on Usage Billing, and each app's admins are told why

## Resources

- `GET /api/v1/resources` - List all resources
- `POST /api/v1/resources` - Create new resource
- `GET /api/v1/resources/{uuid}` - Get resource details, its `cron_minutes` included (see "Cron minutes" below). Returns `region_code`, the value `POST /resources` takes, and `component_quantities`, the components grouped by `code_name` as `[{"code_name": "1gb-ram", "quantity": 2}]`, the shape `POST` and `PUT /resources` accept as `components` (`[]` when the resource has none). The `components` list renders identical units once, so it does not carry counts: send back `component_quantities`, not `components`
- `PUT /api/v1/resources/{uuid}` - Update resource (change plan, add components). An update is priced at today's prices. **A plan or component code ending in `_archived_<date>`** is one retired by a price change: the resources that have it keep it and pay its old price, but it cannot be bought again — sending it here or to `POST /api/v1/resources` is refused with `422`, and for a plan the message names the code of the same size at today's price when there is one. To move such a resource to today's prices, send that code (and component codes from `GET /api/v1/components`); nothing moves it on its own.
- `PATCH /api/v1/resources/{uuid}/labels` - Update resource labels
- `DELETE /api/v1/resources/{uuid}` - Delete the resource and everything on it — applications, services, buckets, stacks — permanently. Takes that resource off the bill; **does not cancel the subscription**, which continues for the workspace plan and any other resources. There is no endpoint that cancels a subscription; see "Stopping the bill"

### Cron minutes

**Every paid resource plan includes 1,000 cron minutes each billing cycle,
counted at the plan's own size and shared by all cron jobs on the resource** — a
resource exempted from billing too, at its plan's list price. A run at another
size counts by its price per minute against the plan's (`GET
/api/v1/apps/{uuid}/cronjobs/sizes`): a $29 2 vCPU / 4 GiB plan's 1,000 minutes
are 5,800 minutes at the $5 1 vCPU / 1 GiB size. Nothing carries over. A plan that
costs nothing includes none: every run of its cron jobs is billed from the first.
On an enterprise contract cron minutes are unlimited unless Miget set a limit per
resource (`cron_minutes_per_resource` under "Limits"). `GET
/api/v1/resources/{uuid}` returns them as `cron_minutes`:

```json
"cron_minutes": {"included": 1000, "used": 388, "remaining": 612}
```

`used` comes from the hours already closed, so the last hour or so is not in it
yet, and it can pass `included`. `cron_minutes` is `null` where none are counted: a
plan that costs nothing, and a contract without a limit.

What happens once they are used up is **each cron job's own `overage`**, set on
`POST` or `PUT /api/v1/apps/{uuid}/cronjobs` (see the cron job endpoints in the
apps reference):

- **`pause`** (the default): the cron job is paused until the next renewal, which
  resumes it; a run already under way finishes. Until then
  `POST .../cronjobs/{id}/resume` is refused with `422`, and one created
  meanwhile starts with `suspended: true`. The workspace admins are emailed at 75%
  and at 100%.
- **`bill`**: it keeps running, and each run past the included minutes is billed
  by the second at its size's price per minute, on Usage Billing. Choosing it needs
  `apps:manage`, and it starts Usage Billing's $5 monthly minimum when the
  subscription pays for nothing else: answered with `422`,
  `{"errors": "...", "checkout_url": "..."}` and the cron job stays as it was —
  send the person to `checkout_url`, then repeat the request. Switched on, it
  resumes the cron job if the used-up minutes paused it, unless the workspace is
  suspended. On a resource exempted from billing this is the only thing that ever
  starts Usage Billing.

On a contract a cron job always pauses at the limit and nothing is billed. Where
there are no minutes, `overage` in the response says what happens whatever was
sent: `bill` where runs are billed (Usage Billing, a plan that costs nothing),
`null` where no run is billed (an enterprise contract without a limit, a resource
exempted from billing on a plan that costs nothing, the workspace pool).

## Projects

- `GET /api/v1/projects` - List all projects
- `POST /api/v1/projects` - Create new project
- `GET /api/v1/projects/{project_id}` - Get project details
- `PUT /api/v1/projects/{project_id}` - Update project
- `DELETE /api/v1/projects/{project_id}` - Delete project
- `GET /api/v1/projects/{project_id}/apps` - List applications in project
- `POST /api/v1/projects/{project_id}/resources` - Assign a resource to the project (body: `resource_id`, optional `with_hosting_projects`). Needs **both** `projects:manage` and `resources:manage`, because assigning narrows who may use the resource. Refused with **422** while the resource still runs workloads the assignment would lock out — pass `with_hosting_projects: true` to assign it to those projects as well instead of being refused
- `DELETE /api/v1/projects/{project_id}/resources/{resource_id}` - Remove the assignment, returning the resource to the shared pool. Needs only `projects:manage`: releasing only ever widens access, and every workload already on the resource stays legal. **422** when that resource is not assigned to that project
- `POST /api/v1/projects/{project_id}/restrictions` - Grant access to the project. Needs `projects:manage`, and you must be able to reach the project yourself. Body carries **exactly one** of `user_email` (a workspace member) or `role_name` (a workspace role — everyone holding it reaches the project); sending both or neither is a **400**. Adding the first entry is what closes an otherwise open project. The **422** replies are distinct, so read the message rather than retrying: `Restricting projects is available on organization and enterprise plans` (checked first, before anything about the subject), `No member of this workspace has the email <address>`, `This workspace has no role named <name>`, `Subject already has access to this project`, and — for a workspace owner who holds no membership row — `The workspace owner already reaches every project`, which means no entry is needed rather than that something failed
- `DELETE /api/v1/projects/{project_id}/restrictions/{id}` - Revoke one entry, where `{id}` is the numeric `id` from the project's `restrictions` list. Removing the last entry reopens the project to the whole workspace

## Project Environment Variables

Addressed by `key`, like app variables; the `id` in the list response is not accepted by any endpoint.

- `GET /api/v1/projects/{project_id}/vars` - List project environment variables
- `POST /api/v1/projects/{project_id}/vars` - Create project environment variable. A key the project already has is refused with `422`
- `PUT /api/v1/projects/{project_id}/vars` - Set project environment variable (identified by `key` in body): overwrites it if it exists, creates it if not
- `DELETE /api/v1/projects/{project_id}/vars` - Delete project environment variable

## Project Secret Files

Shared across every app in the project that has project secret files enabled. Same write-only rule as app secret files. **Creating one changes nothing for an app until that app's `project_files_enabled` toggle is on** — the parallel of `project_variables_enabled` for env vars.

- `GET /api/v1/projects/{project_id}/secret_files` - List project secret files (metadata only)
- `POST /api/v1/projects/{project_id}/secret_files` - Create one (body: `filename`, `text`). Returns the server-assigned `name`
- `GET /api/v1/projects/{project_id}/secret_files/{name}` - Get one file's metadata
- `PUT /api/v1/projects/{project_id}/secret_files/{name}` - Update `filename`, `text`, or both. At least one is required
- `DELETE /api/v1/projects/{project_id}/secret_files/{name}` - Remove it

## Plans, Regions & Components

- `GET /api/v1/plans` - List available plans. Each plan returns `code_name` (the opaque identifier you pass to `POST /api/v1/resources`), `plan_type` (`dev` for development/hobby, `pro` for production), `ram_size` and `disk_size` in **bytes**, `cpu_size` as a fractional core count, and `unit_price` in **cents**. Only plans on sale are listed; a resource's own `plan` (in `GET /api/v1/resources`) may be one archived with a price change, with a code ending in `_archived_<date>` and the `unit_price` it still pays. Sizing note: `ram_size: 268435456` is 256 MiB, not 256 MB — compare it against `quota.ram_size` on apps, which uses the same unit.
- `GET /api/v1/regions` - List available regions. Current regions: `eu-east-1` (Warsaw), `us-east-1` (Vint Hill)
- `GET /api/v1/components` - List available resource components (extra_ram, extra_cpu, extra_disk), only those on sale

## Users

- `GET /api/v1/users/me` - Get current user profile
- `GET /api/v1/users/me/credits` - The user's own credits, referral code and their history. A user collects credits from referrals and posts; they pay for nothing until transferred to a workspace with `POST /api/v1/workspace/credits/transfer`

## SSH Keys

- `GET /api/v1/users/me/ssh_keys` - List SSH keys
- `POST /api/v1/users/me/ssh_keys` - Add an SSH key
- `GET /api/v1/users/me/ssh_keys/{id}` - Get SSH key details
- `DELETE /api/v1/users/me/ssh_keys/{id}` - Remove an SSH key

## Authentication

- `POST /api/v1/auth/sign_in` - Authenticate with email/password
- `POST /api/v1/auth/refresh_token` - Refresh access token

## Create Resource (`POST /api/v1/resources`)

**Required fields:**
- `plan_code_name` (string) - Plan identifier. **Do not guess it** — call `GET /api/v1/plans` and use a `code_name` from the response verbatim. Standard plans are named by their CPU class, CPU and RAM — `free`, `hobby-1c-1g` … `hobby-8c-16g`, `pro-1c-2g` … `pro-48c-96g`, each also a size on Usage Billing — and custom plans have names of their own; none is a friendly word like `"starter"` or `"professional"`. The names standard plans had before (e.g. `miget_hobby_1`, `miget_pro_1`) are still accepted, except `miget_hobby_2`, which was retired with no successor. A wrong value returns a generic 422.
- `region_code` (string) - Deployment region code. Available: `eu-east-1` (Warsaw), `us-east-1` (Vint Hill)

**Optional:**
- `plan_type` (string) - Ignored. The plan is resolved from `plan_code_name` alone; this field is accepted only for backward compatibility. Read `plan_type` off the plan object instead to tell `dev` and `pro` plans apart.
- `components` (array) - Additional resource components (extra RAM, CPU, disk)

**Ask only what you cannot derive:**
- "What plan type? (dev for development, pro for production)"
- "What plan code name? (e.g., free, starter, professional)"
- "Which region? (eu-east-1, us-east-1)"
- "Do you want to add any extra components? (RAM, CPU, disk)"

## Create Project (`POST /api/v1/projects`)

**Required fields:**
- `name` (string) - Project name (must be unique within workspace)

**Optional:**
- `description` (string) - Brief description of the project's purpose

**Ask only what you cannot derive:**
- "What should be the project name?"
- "Do you want to add a description?"

## Permissions & Authorization

Miget uses **role-based access control (RBAC)** within workspaces.

## Common Permissions

- `apps:view` - View applications, their logs and their deployments
- `apps:manage` - Create and delete applications, manage their domains and TLS certificates, and delete or reset a database
- `apps:deploy` - Deploy applications, start, stop and restart them, change their settings, and create git tokens
- `apps:operate` - Add-ons, cron jobs, ports and their public access, environment variables, secret files and settings
- `resources:view` - View resources
- `resources:operate` - Operate resources
- `resources:manage` - Create, update and delete resources
- `projects:view` - View projects
- `projects:operate` - Operate projects
- `projects:manage` - Create, update and delete projects
- `services:view` - View services
- `services:operate` - Operate services
- `services:manage` - Create, update and delete services
- `buckets:view` - View buckets, list objects and download them
- `buckets:operate` - Upload, rename and delete objects, manage policy/ACL, change settings
- `buckets:manage` - Create and delete buckets
- `network:view` - View private networks, their subnets and their attachments
- `network:operate` - Subnets, attachments, peer routes, WireGuard devices, and renaming a VPC
- `network:manage` - Create and delete private networks, VPN gateways and site-to-site tunnels
- `workspace:general` - Manage general workspace settings, API tokens and webhooks
- `workspace:security` - Manage workspace security settings
- `workspace:credentials` - Manage git and registry credentials
- `workspace:integrations` - Manage workspace integrations
- `workspace:billing` - Manage the plan, billing, invoices and orders
- `workspace:members` - Manage workspace members
- `workspace:roles` - Manage workspace roles

Workspace owners have all permissions automatically.

If you get a `403 Forbidden` error, the user doesn't have the required permission for that operation. Let them know which permission is needed so they can request it from a workspace admin.

---
