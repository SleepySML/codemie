# Testing API Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Router tests and router contract guards; suite-wide test patterns are in `testing/testing-patterns.md`.

## Overview

Router tests either call the endpoint function directly with patched collaborators, or build a `TestClient(app)` with `app.dependency_overrides[authenticate]`. The router guards rebuild `main.py` in a subprocess under `PROFILE_ENV` and compare the OpenAPI document and route order with committed baselines.

| Part | What it holds | Start here |
|---|---|---|
| `tests/codemie/rest_api/routers/` | router tests | `tests/codemie/rest_api/routers/test_conversation_metrics_access.py::client` |
| `tests/codemie/rest_api/router_guards/` | OpenAPI and route-order guards | `tests/codemie/rest_api/router_guards/guarded_app.py::load_guarded_app_dump` |

## API tests

### Rules
- R1. Patch a collaborator in the module that looks it up (the endpoint submodule, `_common` only for names `_common` reads); `patch.object` on a class survives module moves. Evidence: `tests/codemie/rest_api/routers/test_projects_router.py::TestProjectCreationEndpoint.test_create_project_calls_service_directly`
- R2. Test unauthenticated and unauthorized calls for a protected route and assert the structured `error` body fields, not raw exception text. Evidence: `tests/codemie/rest_api/routers/test_conversation_metrics_access.py::test_the_route_requires_an_authenticated_caller`
- R3. Fix the code when the OpenAPI or route-order guard fails on an unintended change; regenerate baselines with `regenerate_baselines` after a reviewed change, and add a new flag-gated router's flag to `PROFILE_ENV`. Evidence: `tests/codemie/rest_api/router_guards/guarded_app.py::PROFILE_ENV`

## Examples

### A TestClient fixture with an auth override

Source: `tests/codemie/rest_api/routers/test_conversation_metrics_access.py::client`

```python
def client():
    app.dependency_overrides[authenticate] = lambda: User(id="caller", name="Caller")
    yield TestClient(app)
    app.dependency_overrides.clear()
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| The OpenAPI guard fails after an unrelated-looking change | any change to a path, operationId, tag or schema changes the snapshot | fix the code if the change was unintended; else regenerate and review the baseline diff | `tests/codemie/rest_api/router_guards/test_openapi_snapshot.py::test_openapi_matches_baseline` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Authenticate in a TestClient test | override `authenticate` and clear overrides after | `tests/codemie/rest_api/routers/test_conversation_metrics_access.py::client` |
| Regenerate guard baselines | run the regenerate module after a reviewed contract change | `tests/codemie/rest_api/router_guards/regenerate_baselines.py::main` |
| Guard a flag-gated router | add its flag to `PROFILE_ENV` | `tests/codemie/rest_api/router_guards/guarded_app.py::PROFILE_ENV` |
