# External Services

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Adapter boundary, credentials, provider errors and inbound webhooks; model providers are in `integration/llm-providers.md`.

## Overview

External systems are reached through `codemie_tools` toolkits (agent-callable) and `src/codemie/datasource/` processors (indexing). Inbound webhooks enter `WebhookService`, which verifies GitHub signatures, GitLab tokens or a legacy header and decides whether to dispatch.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie_tools/` | toolkits per external system | `src/codemie_tools/base/toolkit_provider.py::is_toolkit_class` |
| `src/codemie/datasource/` | datasource processors and loaders | `src/codemie/datasource/base_datasource_processor.py::BaseDatasourceProcessor` |
| `src/codemie/triggers/bindings/` | inbound webhook verification and cron triggers | `src/codemie/triggers/bindings/webhook.py::WebhookService.verify_security_header` |

## External services

### Rules
- R1. Give authentication failures an `error_cause` metric attribute (`gitlab_token_invalid`, `github_signature_verification_failed`, `invalid_security_header`); a filtered GitLab MR event is a success metric with `filtered_action`. Evidence: `src/codemie/triggers/bindings/webhook.py::WebhookService._verify_gitlab_token`
- R2. Call external SDKs from a service, datasource processor or toolkit adapter and normalize provider objects at that boundary. Evidence: `src/codemie/datasource/base_datasource_processor.py::BaseDatasourceProcessor`
- R3. Map provider exceptions to typed errors with fixed messages and retry only classified transient errors within a bounded budget, as the Exchange client does. Evidence: `src/codemie_tools/ms_exchange/write_retry.py::run_write`
- R4. Return True from `verify_security_header` to dispatch and False to acknowledge a filtered event with 200; raise only for an authentication failure, since providers deactivate endpoints after repeated non-2xx. Evidence: `src/codemie/triggers/bindings/webhook.py::WebhookService.verify_security_header`
- R5. Keep provider-specific verification in a `*_webhook_security.py` class and parse its body without raising; `webhook.py` owns the logging and metrics. Evidence: `src/codemie/triggers/bindings/gitlab_webhook_security.py::GitLabWebhookSecurity.verify_and_filter`

## Examples

### Parsing a webhook body without raising

Source: `src/codemie/triggers/bindings/gitlab_webhook_security.py::GitLabWebhookSecurity._parse_body`

```python
    def _parse_body(cls, body: bytes | str | dict[str, object]) -> dict[str, object]:
        """Best-effort parse of a webhook body into a dict; return ``{}`` on failure.

        A parse failure or non-dict payload is treated as "not a merge_request event"
        so downstream filter logic can pass it through unchanged.
        """
        if isinstance(body, dict):
            return body
        if isinstance(body, (bytes, bytearray)):
            try:
                body = body.decode("utf-8", errors="strict")
            except UnicodeDecodeError:
                return {}
        if isinstance(body, str):
            if not body:
                return {}
            try:
                parsed = json.loads(body)
            except (json.JSONDecodeError, TypeError):
                return {}
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A webhook with no security settings dispatches each request | `_handle_no_security` logs a warning and returns True for backward compatibility | configure a GitHub secret, GitLab token or security header on the webhook | `src/codemie/triggers/bindings/webhook.py::WebhookService._handle_no_security` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a webhook provider | add a `*_webhook_security.py` class and a branch in `verify_security_header` | `src/codemie/triggers/bindings/webhook.py::WebhookService.verify_security_header` |
| Add a datasource | subclass `BaseDatasourceProcessor` and implement `_index_name`, `_init_loader`, `_init_index` | `src/codemie/datasource/base_datasource_processor.py::BaseDatasourceProcessor` |
| Pre-fill an integration URL for a deployment | set it under `tool_defaults` in the customer config | `README.md` § Customer Configuration |
