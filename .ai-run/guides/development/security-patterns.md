# Security Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Authentication, secrets and untrusted input in application code; CVE remediation is in `security/README.md`.

## Overview

`authenticate` is the route dependency for both the internal bind-key flow (HMAC over nonce, timestamp and user id) and the external IdP flow, and it stores the user in request state and logging context. Role checks are separate dependencies such as `admin_access_only` and access helpers such as `project_access_check`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/rest_api/security/authentication.py` | authentication and role dependencies | `src/codemie/rest_api/security/authentication.py::authenticate` |
| `src/codemie/triggers/bindings/` | inbound webhook verification | `src/codemie/triggers/bindings/gitlab_webhook_security.py::GitLabWebhookSecurity.verify_token` |

## Security

### Rules
- R1. Protect routes with `authenticate` and the role dependencies; do not parse bearer or bind-key headers in an endpoint. Evidence: `src/codemie/rest_api/security/authentication.py::authenticate`
- R2. Keep credentials in `Config` fields or integration settings, and list new secret keys in `LIST_OF_SENSITIVE_FIELDS`; do not log tokens. Evidence: `src/codemie/service/settings/settings.py::LIST_OF_SENSITIVE_FIELDS`
- R3. Parse external input strictly (`errors="strict"`) and treat what does not parse as unknown, without lossy repair before a security decision. Evidence: `src/codemie/triggers/bindings/gitlab_webhook_security.py::GitLabWebhookSecurity._parse_body`
- R4. Check a named USER grant with `grant_exists` (or `live_grants`) from `permission.py`, passing the type's recipient policy such as `eligible_internal_user` (it joins `users` and is empty while `ENABLE_USER_MANAGEMENT` is off); `Permission.get_for` reads APPLICATION grants only. Evidence: `src/codemie/rest_api/models/permission.py::grant_exists`

### Constraints
| Do not | Do instead | Why | Evidence |
|---|---|---|---|
| Protect a read route over an owner-only view of a resource (its grantees, a sharing search) with `authenticate` alone | Go through a service method that checks the owner before reading, as `AssistantShareService.list_for_owner` does | any signed-in user could list another owner's grantees or search the user directory | `src/codemie/service/assistant/assistant_share_service.py::AssistantShareService.list_for_owner` |

## Examples

### A role dependency that logs and raises 403

Source: `src/codemie/rest_api/security/authentication.py::admin_access_only`

```python
async def admin_access_only(request: Request):
    """
    Checks if current user is admin or maintainer
    """
    if not request.state.user.is_admin_or_maintainer:
        logger.warning(f"access_denied_admin: actor_user_id={request.state.user.id}, domain=user_management")
        raise ExtendedHTTPException(
            code=status.HTTP_403_FORBIDDEN,
            message=ACCESS_DENIED_MESSAGE,
            details=_ADMIN_OR_MAINTAINER_REQUIRED,
            help=_ELEVATED_ACCESS_HELP,
        )
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| An internal service call fails with 401 although the bind key is right | the bind-key path requires `X-Bind-Nonce`, `X-Bind-Timestamp` and the user-id header together | sign the request with `sign_internal_request` | `src/codemie/rest_api/security/authentication.py::sign_internal_request` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Protect a route | add `Depends(authenticate)` or put it on the router | `src/codemie/rest_api/security/authentication.py::authenticate` |
| Restrict to admins or maintainers | add `admin_access_only` | `src/codemie/rest_api/security/authentication.py::admin_access_only` |
| Check project access in a handler | call `project_access_check` | `src/codemie/rest_api/security/authentication.py::project_access_check` |
