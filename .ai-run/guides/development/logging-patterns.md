# Logging Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Contextual logging, severity and metric failure labels; exception types and handlers are in `development/error-handling.md`.

## Overview

Logging context (request UUID, user id, conversation id, email) lives in ContextVars set by `set_logging_info`; the request middleware sets the UUID from `X-Request-ID` and `authenticate` adds the user. Work forked into another thread or task carries the context through `copy_logging_context` and `restore_logging_context`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/configs/logger.py` | log config, record factory, context helpers | `src/codemie/configs/logger.py::set_logging_info` |
| `src/codemie/rest_api/main.py` | per-request logging middleware | `src/codemie/rest_api/main.py::configure_logging` |
| `src/codemie/triggers/bindings/webhook.py` | verification metrics with per-cause labels | `src/codemie/triggers/bindings/webhook.py::WebhookService._send_verification_metric` |

## Logging

### Rules
- R1. Log operation, ids and status with sanitized detail, never tokens or raw auth headers; request and user context come from `set_logging_info`. Evidence: `src/codemie/configs/logger.py::set_logging_info`
- R2. Give a security failure (token, signature) its own metric `error_cause`, separate from benign filtering, which the GitLab path reports as a success metric with `filtered_action`. Evidence: `src/codemie/triggers/bindings/webhook.py::WebhookService._verify_gitlab_token`
- R3. Log expected client errors at warning and server failures at error with the stack, as the `ExtendedHTTPException` handler does. Evidence: `src/codemie/rest_api/main.py::extended_http_exception_handler`

## Examples

### Restoring logging context inside a forked task

Source: `src/codemie/configs/logger.py::restore_logging_context`

```python
def restore_logging_context(snapshot: LoggingContextSnapshot) -> None:
    """
    Restore logging ContextVar values captured by copy_logging_context().
    Must be called at the start of a forked task/thread, before any logging.
    """
    set_logging_info(
        uuid=snapshot["uuid"],
        user_id=snapshot["user_id"],
        conversation_id=snapshot["conversation_id"],
        user_email=snapshot["user_email"],
    )
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| Log lines from a thread pool or hedged task show `-` for user and request | ContextVars set in the request do not reach a new thread | capture with `copy_logging_context` and call `restore_logging_context` first in the task, as the hedged handler does | `src/codemie/rest_api/handlers/hedged_handler.py::copy_logging_context` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Bind a conversation id to later logs | call `set_logging_info(conversation_id=...)` | `src/codemie/configs/logger.py::set_logging_info` |
| Carry log context into a thread | snapshot with `copy_logging_context` | `src/codemie/configs/logger.py::copy_logging_context` |
| Correlate logs with a client request | send `X-Request-ID`; the middleware reuses it | `src/codemie/rest_api/main.py::configure_logging` |
