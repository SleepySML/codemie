# Error Handling

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Typed exceptions, central handlers and accurate messages; log severity and metric labels are in `development/logging-patterns.md`.

## Overview

Typed exceptions live in `src/codemie/core/exceptions.py`; `main.py` maps `ValidationException` to 400, `NotFoundException` to 404 and `ExtendedHTTPException` to its own code, and a middleware turns any other exception into a 500 with a request-id message outside local runs.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/core/exceptions.py` | shared exception types | `src/codemie/core/exceptions.py::ExtendedHTTPException` |
| `src/codemie/rest_api/main.py` | handlers and the catch-all middleware | `src/codemie/rest_api/main.py::handle_exceptions` |

## Key abstractions

| Abstraction | What it is | Where |
|---|---|---|
| ValidationException | a `ValueError` subclass mapped to 400 with `str(exc)` as the message | `src/codemie/core/exceptions.py::ValidationException` |
| ExtendedHTTPException | carries code, message, details and help into the error body | `src/codemie/core/exceptions.py::ExtendedHTTPException` |

## Errors

### Rules
- R1. Raise `ValidationException` (400), `NotFoundException` (404) or `ExtendedHTTPException` for user-facing errors; a bare `ValueError` raised in a service or route reaches the client as a 500 (one raised inside a request-model `field_validator` becomes a 422 through `validation_exception_handler`). Evidence: `src/codemie/rest_api/main.py::domain_validation_exception_handler`
- R2. Log the exception type and detail server-side and return a fixed message, as `authenticate` does for unexpected errors. Evidence: `src/codemie/rest_api/security/authentication.py::authenticate`
- R3. Give each failure mode its own message or metric cause instead of one shared message, and check a value before interpolating it into a message. Evidence: `src/codemie/triggers/bindings/webhook.py::WebhookService.invoke_webhook_logic`

## Examples

### Mapping an exception type to an error body

Source: `src/codemie/rest_api/main.py::domain_validation_exception_handler`

```python
async def domain_validation_exception_handler(request: Request, exc: ValidationException) -> JSONResponse:
    return JSONResponse(
        status_code=status.HTTP_400_BAD_REQUEST,
        content={"error": {"message": str(exc), "details": None, "help": None}},
    )
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A plain `ValueError` raised from model validation becomes a 500 | only `ValidationException` has a 400 handler; `BaseModelWithSQLSupport.save` raises plain `ValueError` | catch it in the service and raise `ValidationException` | `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport.save` |
| A validation message leaks internal text to the client | the 400 handler returns `str(exc)` as the message | write the message for the client and log the detail | `src/codemie/rest_api/main.py::domain_validation_exception_handler` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Return 404 | raise `NotFoundException` or `ExtendedHTTPException(code=404, ...)` | `src/codemie/rest_api/main.py::not_found_exception_handler` |
| Return 400 for bad input | raise `ValidationException` | `src/codemie/core/exceptions.py::ValidationException` |
| Return a structured error with help text | raise `ExtendedHTTPException` with `help` | `src/codemie/core/exceptions.py::ExtendedHTTPException` |
