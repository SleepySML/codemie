# Endpoint Conventions

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Request and response models and PUT and POST semantics; router registration, errors and auth are in `api/rest-api-patterns.md`.

## Overview

Request and response schemas live in `src/codemie/rest_api/models/` so routers and services import one definition. Shared router helpers sit in `routers/utils.py` or a package-private `_common.py`, and route semantics for PUT and POST are pinned by router tests.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/rest_api/models/` | request and response models | `src/codemie/rest_api/models/budget.py::BudgetCreateRequest` |
| `src/codemie/rest_api/routers/utils.py` | helpers shared across router modules | `src/codemie/rest_api/routers/utils.py::get_assistant_by_id_or_raise` |
| `src/codemie/rest_api/routers/projects/_common.py` | helpers private to one router package | `src/codemie/rest_api/routers/projects/_common.py::_authorize_project_access` |

## Endpoint conventions

### Rules
- R1. Define a schema that a service or another module imports in `src/codemie/rest_api/models/<feature>.py`, not in the router module. Evidence: `src/codemie/service/budget/budget_service.py::BudgetCreateRequest`
- R2. Keep routers to validation and delegation; share helpers across routers in `routers/utils.py` and helpers of one router package in a package-private module such as `_common.py`, never importing another router module's `_helper`. Evidence: `src/codemie/rest_api/routers/projects/_common.py::_authorize_project_access`
- R3. Implement PUT as full replace: re-sending the same body leaves the entity unchanged apart from update timestamps; the idempotent-write tests pin it. Evidence: `tests/codemie/rest_api/routers/test_idempotent_write.py::TestAssistantIdempotent`
- R4. Expect assistants and workflows to create a new entity on each POST without an id; skills and categories reject a duplicate name with 409. Evidence: `tests/codemie/rest_api/routers/test_create_always_new.py::TestPostCreatesNewEntity`

## Examples

### A shared lookup helper that raises a typed 404

Source: `src/codemie/rest_api/routers/utils.py::get_assistant_by_id_or_raise`

```python
def get_assistant_by_id_or_raise(assistant_id: str) -> Assistant:
    """
    Retrieves an assistant by ID or raises a standardized exception if not found
    """
    assistant = Assistant.find_by_id(assistant_id)
    if not assistant:
        raise ExtendedHTTPException(
            code=status.HTTP_404_NOT_FOUND,
            message="Assistant not found",
            details=f"No assistant found with the id '{assistant_id}'.",
            help="Please check the assistant id and ensure it is correct. ",
        )
    return assistant
```

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a request model | create or extend `src/codemie/rest_api/models/<feature>.py` | `src/codemie/rest_api/models/budget.py::BudgetCreateRequest` |
| Share a helper across routers | add it to the routers utils module | `src/codemie/rest_api/routers/utils.py::get_assistant_by_id_or_raise` |
| Share a helper inside one router package | put it in the package's `_common.py` | `src/codemie/rest_api/routers/projects/_common.py::_authorize_project_access` |

## Create Semantics (POST)

Assistants and workflows create a new entity on every POST with no id, even when an identical one exists. Skills and categories reject a duplicate name with 409; project and user settings reject a duplicate alias with 422. Pinned by `tests/codemie/rest_api/routers/test_create_always_new.py`.
