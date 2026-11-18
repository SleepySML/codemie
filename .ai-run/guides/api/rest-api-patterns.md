# REST API Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: FastAPI router registration, error responses and auth dependencies; request models and PUT/POST semantics are in `api/endpoint-conventions.md`.

## Overview

Routers live under `src/codemie/rest_api/routers/`; a large router is a package whose `__init__.py` builds the parent `APIRouter` with tags, prefix, the `authenticate` dependency and, for an LLM-backed package, `require_core`, and includes its submodules in order. Errors leave through exception handlers and a fallback middleware in `main.py`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/rest_api/routers/` | route modules and router packages | `src/codemie/rest_api/routers/conversation/__init__.py::router` |
| `src/codemie/rest_api/main.py` | router registration, middleware, exception handlers | `src/codemie/rest_api/main.py::handle_exceptions` |
| `src/codemie/rest_api/security/` | authentication and role dependencies | `src/codemie/rest_api/security/authentication.py::authenticate` |
| `src/codemie/rest_api/handlers/` | request handlers for assistant chat | `src/codemie/rest_api/handlers/assistant_handlers.py::get_request_handler` |

## Routers

### Rules
- R1. Register a router with `app.include_router` in `main.py`; a large router becomes a package whose `__init__.py` builds the parent `APIRouter` (tags, prefix, `authenticate` and, for LLM-backed routers, `require_core`) and includes each submodule's bare `router`. Evidence: `src/codemie/rest_api/routers/conversation/__init__.py::router`
- R2. Keep a literal path ahead of a parameterized one when ordering includes (Starlette serves the first match); the route-precedence guard fails when a recorded pair swaps. Evidence: `tests/codemie/rest_api/router_guards/test_route_precedence.py::test_baseline_order_is_preserved`
- R3. Raise `ValidationException`, `NotFoundException` or `ExtendedHTTPException` and let the handlers in `main.py` build the error body; pass client-safe `details` on any 5xx except an unexpected 500 (`expected=False`), which the handler returns as given. Evidence: `src/codemie/rest_api/main.py::extended_http_exception_handler`
- R4. Depend on `authenticate` and the role dependencies (`admin_access_only` and siblings) instead of parsing auth headers in an endpoint. Evidence: `src/codemie/rest_api/security/authentication.py::admin_access_only`
- R5. Add `Depends(require_core)` to a router or endpoint that needs the LLM-backed core: in STANDALONE mode without a core provider it answers 404 and `schema_routes` hides the route from the OpenAPI schema; `test_every_core_route_is_guarded` fails only for an unguarded route under `/v1/assistants`, `/v1/workflows`, `/v1/conversations`, `/v1/skills` or `/v1/ide/inference`, so a route elsewhere relies on the author. Evidence: `src/codemie/rest_api/dependencies/core.py::require_core`

## Examples

### A router package that includes its submodules in order

Source: `src/codemie/rest_api/routers/conversation/__init__.py::router`

```python
router = APIRouter(
    tags=["Conversation"],
    prefix="/v1",
    dependencies=[Depends(require_core), Depends(authenticate)],
)
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A 503 or 502 from an `ExtendedHTTPException` shows internal details to the client | the handler replaces `details` with the request-id message only for an unexpected 500 (`expected=False`); other 5xx codes and any `expected=True` 5xx return `exc.details` as given | raise 500 for internal faults, or pass client-safe `details` | `src/codemie/rest_api/main.py::extended_http_exception_handler` |
| An unhandled exception surfaces as a raw traceback locally but a generic 500 in other environments | `handle_exceptions` re-raises when `config.is_local` | reproduce error-body behaviour with a non-local `ENV` | `src/codemie/rest_api/main.py::handle_exceptions` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add an endpoint to an existing package | add it to the submodule's bare `router` | `src/codemie/rest_api/routers/projects/crud.py::router` |
| Return a client error | raise `ExtendedHTTPException` with code, message, details and help | `src/codemie/core/exceptions.py::ExtendedHTTPException` |
| Require an admin or maintainer | add the `admin_access_only` dependency | `src/codemie/rest_api/security/authentication.py::admin_access_only` |
