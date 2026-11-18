# Provisioning API

`/v1/provisioning` serves third-party systems (provisioning clients) that apply an approved business request to CodeMie in one call. It follows `.ai-run/guides/api/endpoint-conventions.md`; this guide states only where it departs from it and the rules every provisioning endpoint adds.

## Overview

`mount_provisioning_api` mounts `/v1/provisioning` only when the API and user management are both enabled; its parent router authenticates, admits registered provisioning clients alone and records its routes as their allowlist. `projects.py` lists the shared projects of given people, and `budgets.py` sets, adjusts and cancels a project's budget through `ProvisioningBudgetService`, which runs a read-only check phase and then two commits.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/rest_api/routers/provisioning/__init__.py` | parent router, guards, mounting | `src/codemie/rest_api/routers/provisioning/__init__.py::mount_provisioning_api` |
| `src/codemie/rest_api/routers/provisioning/projects.py` | member project lookup | `src/codemie/rest_api/routers/provisioning/projects.py::list_member_projects` |
| `src/codemie/rest_api/routers/provisioning/budgets.py` | budget routes, descriptions, documented responses | `src/codemie/rest_api/routers/provisioning/budgets.py::set_project_budget` |
| `src/codemie/rest_api/security/provisioning_access.py` | client guard, permission dependency, scope lock | `src/codemie/rest_api/security/provisioning_access.py::RequirePermission` |
| `src/codemie/rest_api/models/provisioning.py` | request and response models | `src/codemie/rest_api/models/provisioning.py::ProvisioningBudgetRequest` |
| `src/codemie/service/provisioning_budget_service.py` | check phase, commit 1, commit 2, partly-applied answers | `src/codemie/service/provisioning_budget_service.py::ProvisioningBudgetService._upsert` |
| `src/codemie/service/provisioning_service.py` | project lookup and creation, cost center link, admin assignment | `src/codemie/service/provisioning_service.py::ProvisioningService.resolve_project` |
| `src/codemie/configs/provisioning_clients_config.py` | client registry and permissions | `src/codemie/configs/provisioning_clients_config.py::ProvisioningClientsConfig.load` |

## Namespace And Access

| Rule | Where |
|---|---|
| Every route lives under `/v1/provisioning`, in a resource sub-router of `routers/provisioning/` included by the parent router | `src/codemie/rest_api/routers/provisioning/__init__.py::router` (`router.include_router(...)`) |
| The parent router applies `authenticate` then `provisioning_clients_only`; its routes are also the clients' allowlist, so a route added under it is reachable by clients and a client gets 403 anywhere else | `src/codemie/rest_api/security/provisioning_access.py::enforce_provisioning_scope` (`provisioning_clients_only`, `set_provisioning_routes`) |
| The router is mounted only when `PROVISIONING_API_ENABLED` and `ENABLE_USER_MANAGEMENT` are both true | `src/codemie/rest_api/routers/provisioning/__init__.py::mount_provisioning_api` |

## Client Registry

| The external system | Registry |
|---|---|
| Calls `/v1/provisioning` with a token of its own CodeMie user | Provisioning client registry: `config/provisioning_clients/provisioning-clients-config.yaml` (`provisioning_clients_config`, keyed by CodeMie user id) |
| Signs its requests with a public key and acts on `allowed_resources` | Authorized applications: `config/authorized_applications/authorized-applications-config.yaml` (`authorized_applications_config`) |

A system calling `/v1/provisioning` always goes in the provisioning client registry, never in authorized applications. A listed client is confined to this API.

| Permission | Lets the client | Grant |
|---|---|---|
| `provisioning:read-projects` | List the shared projects of given people | Any client that needs the lookup |
| `provisioning:manage-projects` | Create shared projects and make any active user an admin of any existing shared project | Only to a client trusted with that reach |
| `provisioning:manage-budgets` | Replace or cancel the budget of every shared project, including one a maintainer set | Only to a client trusted with that reach |

A client that creates projects with budgets needs both management permissions. A new permission is a `ProvisioningPermission` member and a line in the registry file's header comment. Evidence: `src/codemie/configs/provisioning_clients_config.py::ProvisioningPermission`, `src/codemie/service/provisioning_budget_service.py::ProvisioningBudgetService._check_project`

A test that needs a registered provisioning client builds it with `make_client(name, user_id, permissions)` and registers it with `with registered_clients(client):`, which patches `provisioning_clients_config.clients` for the context; never edit `provisioning-clients-config.yaml` for a test (registry loader tests write their own file under `tmp_path`). Evidence: `tests/codemie/configs/provisioning_clients_support.py::registered_clients`, `tests/codemie/configs/provisioning_clients_support.py::make_client`

## Permission Checks

| Rule | Where |
|---|---|
| A route declares exactly one permission: `Depends(RequirePermission(...))` | `src/codemie/rest_api/routers/provisioning/budgets.py::set_project_budget`; pinned by `tests/codemie/rest_api/routers/test_provisioning_router.py::TestRouteContract.test_every_provisioning_route_declares_one_permission_after_guards` |
| A permission the target state triggers is checked with `require_permission(client, permission, actor_user_id)` in the service's check phase, before anything is written. Never add a second dependency or a second 403 path | `src/codemie/service/provisioning_budget_service.py::ProvisioningBudgetService._check_project` requires `manage-projects` when the project is missing or the `project` block names a cost center or admins |
| The route's 403 description names every condition that can produce it | `src/codemie/rest_api/routers/provisioning/budgets.py::_WRITE_RESPONSES` |

## Paths, Bodies And Methods

| Rule | Notes |
|---|---|
| Natural keys in paths, never internal ids | `/projects/{project_name}/budget`; the name is used exactly as sent, 3-100 characters, else 422 (`ProvisioningProjectName`). Names are never trimmed or case-folded before a permission or lookup decision (`security-patterns.md`). Emails are `EmailStr`, which returns email-validator's normalized address: surrounding spaces and a `Name <...>` display name are dropped and the domain is lower-cased, while the local part keeps its case (except RFC 2142 role names such as `postmaster`). Case variants of one email count once, as first sent (`ProvisioningProjectBlock`) |
| CodeMie terms in bodies, descriptions and help texts | No vendor names, no caller-specific fields; the caller maps its own fields |
| Body = target state plus an optional add-only prerequisite block | `amounts` is the target; `project` (`cost_center`, `admin_emails`) only adds what the target needs |
| Unknown body fields are rejected with 422 | `ConfigDict(extra="forbid")` on `src/codemie/rest_api/models/provisioning.py::ProvisioningBudgetRequest` and `ProvisioningProjectBlock`; a misspelt `admin_emails` would otherwise skip admins silently |
| 422 is documented in the platform's validation-error shape | `src/codemie/rest_api/models/provisioning.py::ProvisioningValidationErrorResponse` |

| Method | Meaning | Status |
|---|---|---|
| PUT | Set the target state. Creates the parent (the project) when missing | 201 when the call created the target, 200 when it replaced one |
| PATCH | Adjust: merge into the target, creating what is missing | 200 |
| DELETE | Cancel the target. Never creates; no target is not an error | 204; 404 when the parent is missing |
| POST | Not used for writes | |

Departures from `endpoint-conventions.md` (Update Semantics (PUT)): PUT creates the parent, the prerequisite block is add-only, PUT answers 201 or 200, and unknown fields are rejected (generic request models ignore them).

Admins in a prerequisite block are only ever added, never removed or replaced. People without an active, non-service-account user are reported as `admins_not_assigned` and are added when the request is sent again after they sign in. A future endpoint whose target is the project itself must say in its description whether its admin list replaces or adds.

A cost center in a prerequisite block is linked only while the project has no active one: a link to a deleted cost center counts as no link, so the named cost center is linked, while a link to an active cost center never changes.

## Change Reference

Every write carries the required `X-Change-Reference` header (1-64 characters; missing or empty is 422). It is recorded only in log lines (`change_reference=` next to `client=`, `domain=provisioning`) and in the name of a newly created budget (`<client name> <reference>`); a client name has at most 35 characters, so the 100-character name always holds the whole reference. It is not in the activity history. Evidence: `src/codemie/rest_api/routers/provisioning/budgets.py::ChangeReference`, `src/codemie/service/provisioning_budget_service.py::_create_request`

## All-Or-Nothing Contract

A write cannot be one transaction because it also syncs an external provider, so the contract is made of ordering and resending.

| Phase | Does | Failure answers |
|---|---|---|
| Check (read-only) | Raises every deterministic 4xx: lookup, permissions, names, cost center, budget rules | 4xx; nothing changed |
| Commit 1 | Project, cost center, admins; find-or-create | 5xx |
| Commit 2 | The target (budget group) and the provider sync | A 4xx or an unexpected error here becomes 500 `partly applied; send the same request again`; a provider failure during the write passes the provider's own 502 through; a category left unsynced after the commit is 502 `partly applied; send the same request again` |

| Rule | Where |
|---|---|
| Every 4xx means nothing changed. 5xx or no answer means send the same request again | Every write route's description says so |
| A resend converges because every write is find-or-create. The resend may answer 200 instead of 201 and `created_project: false` | `src/codemie/service/provisioning_service.py::ProvisioningService.resolve_project`, `src/codemie/service/provisioning_budget_service.py::ProvisioningBudgetService._commit_budget` |
| A check that depends on commit 1 cannot answer 4xx. Put each deterministic check in the check phase; anything after commit 1 answers 5xx | `src/codemie/service/provisioning_budget_service.py::ProvisioningBudgetService._check`, `src/codemie/service/provisioning_budget_service.py::ProvisioningBudgetService._commit_budget_after_project` |
| The cause of a 5xx is logged with `client`, `change_reference` and the project; the 500 body carries only the generic message | `src/codemie/service/provisioning_budget_service.py::_log_partly_applied` |

## Versioning

Within `/v1` a contract change is additive only: a new route, a new optional request field, a new response field, a new permission. Never remove, rename or retype a field, tighten validation, or change a status code; that needs a new version. The OpenAPI baseline changes with any of it: `tests/codemie/rest_api/router_guards/test_openapi_snapshot.py::test_openapi_matches_baseline` fails until the baseline is regenerated and its diff reviewed.

Evidence: `src/codemie/rest_api/routers/provisioning/budgets.py` (routes, descriptions, responses), `src/codemie/rest_api/models/provisioning.py` (models), `src/codemie/service/provisioning_budget_service.py` (check phase, commits, partly-applied 5xx), `src/codemie/service/provisioning_service.py` (project steps), `src/codemie/configs/provisioning_clients_config.py` (registry, permissions).

## Examples

### A permission the target state triggers, checked in the read-only check phase

Source: `src/codemie/service/provisioning_budget_service.py::ProvisioningBudgetService._check_project`

```python
    def _check_project(call: _Call, block: ProvisioningProjectBlock) -> Application | None:
        with get_session() as session:
            project = ProvisioningService.find_project(session, call.project_name)
            if project is None or block.cost_center is not None or block.admin_emails:
                require_permission(call.client, ProvisioningPermission.MANAGE_PROJECTS, call.actor.id)
            ProvisioningService.check_project_prerequisites(session, project, call.project_name, block.cost_center)
        return project
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A validation error answers 500 `partly applied` instead of its 4xx | the check runs after commit 1, where `_commit_budget_after_project` turns a 4xx or an unexpected error into 500 | raise it in the check phase (`_check`, `_check_project`) | `src/codemie/service/provisioning_budget_service.py::ProvisioningBudgetService._commit_budget_after_project` |
| A budget write answers 502 although commit 2 stored the budget | a category is not synced with the active provider, or a member push failed | send the same request again; the resend updates the existing group | `src/codemie/service/provisioning_budget_service.py::ProvisioningBudgetService._commit_budget` |
| The provisioning client registry does not load | an entry is invalid: a name over `CLIENT_NAME_MAX_LENGTH` (it would cut the change reference off the budget name), a non-UUID `user_id`, a duplicate, or the Teams service account | correct the entry; the error names its index and name | `src/codemie/configs/provisioning_clients_config.py::ProvisioningClientsConfig.load` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a provisioning route | add it to a resource sub-router included by the parent router and declare one `RequirePermission` | `src/codemie/rest_api/routers/provisioning/__init__.py::router` |
| Add a permission | add a `ProvisioningPermission` member and list it in the registry file's header comment | `src/codemie/configs/provisioning_clients_config.py::ProvisioningPermission` |
| Add a deterministic check to a budget write | raise it in the check phase, before commit 1, so its 4xx means nothing changed | `src/codemie/service/provisioning_budget_service.py::ProvisioningBudgetService._check` |