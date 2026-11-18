# CLI Analytics Events

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Contract of the CLI plugin events on the ingest routes; storage, windows and money are in `integration/cli-analytics-storage.md`.

## Overview

The CLI analytics ingest routes take hook events as NDJSON on `event-hooks` and OTLP logs, metrics and traces on their own routes. Each request passes pre-body checks, is gated by the legacy project gate or the evidence gate that `IngestRoute` builds from its headers, and is then stored by the PostgreSQL ingestor or discarded as a whole with HTTP 200. `classification-policy` serves the corporate organization list and its version.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/rest_api/routers/cli_analytics.py` | ingest routes, error-to-status mapping, classification policy, read routes | `src/codemie/rest_api/routers/cli_analytics.py::ingest_event_hooks` |
| `src/codemie/rest_api/routers/cli_analytics_ingest_route.py` | evidence headers, organization envelope, publication contract, choice of gate | `src/codemie/rest_api/routers/cli_analytics_ingest_route.py::IngestRoute` |
| `src/codemie/repository/cli_analytics/ports.py` | ingest ports, the gate protocol, session binding, ingest errors | `src/codemie/repository/cli_analytics/ports.py::IngestGate` |
| `src/codemie/repository/cli_analytics/postgres/otlp.py` | decoding hook events and OTLP into rows, event identity | `src/codemie/repository/cli_analytics/postgres/otlp.py::decode_hook_events` |
| `src/codemie/repository/cli_analytics/postgres/ingestor.py` | session checks, gate call, store or discard answer | `src/codemie/repository/cli_analytics/postgres/ingestor.py::PostgresTelemetryIngestor.ingest_hook_events` |
| `src/codemie/repository/cli_analytics/vocabulary.py` | event types and the hook fields with typed columns | `src/codemie/repository/cli_analytics/vocabulary.py::HOOK_TEXT_FIELDS` |

## CLI analytics events

### Rules
- R1. Accept CLI events on `/v1/analytics/cli-analytics/event-hooks` (NDJSON) and the OTLP `logs`, `metrics` and `traces` routes; the sender comes from the token. Evidence: `src/codemie/rest_api/routers/cli_analytics.py::ingest_event_hooks`
- R2. Decide store or discard through the `IngestGate` before anything is written; a discarded body answers 200, because the CLI deletes its spool on any 2xx. Evidence: `src/codemie/repository/cli_analytics/ports.py::IngestGate`
- R3. Serve the corporate organization list from `classification-policy` and classify sessions on the server from the routing and organization headers; the organization name is not stored. Evidence: `src/codemie/rest_api/routers/cli_analytics.py::classification_policy`
- R4. Send one JSON object per NDJSON line with the envelope `type`, `timestamp`, `session_id`, `prompt_id`; a field with a typed column carries that column's name from `HOOK_TEXT_FIELDS`, and a key without one is kept in the residual attributes. Evidence: `src/codemie/repository/cli_analytics/vocabulary.py::HOOK_KNOWN_KEYS`
- R5. Deduplicate an event with an `event_id` on `(type, session_id, event_id)`; one without it is identified by its whole content. Evidence: `src/codemie/repository/cli_analytics/postgres/otlp.py::event_id`
- R6. Show a private session to its owner alone and decide which subscription classes are private through `scope.private_classes` (legacy: personal; contract without work consent: personal and unknown; with consent: none); `personal_only` filters the sessions list. Evidence: `src/codemie/rest_api/routers/cli_analytics.py::personal_only`, `src/codemie/repository/cli_analytics/scope.py::private_classes`
- R7. Split the event contract so each guide stays under the 400-line cap.
- R8. Keep one request within `ANALYTICS_INGEST_MAX_BODY_BYTES` (checked in `ingest_event_hooks` and `_ingest_otlp`) and `MAX_SESSIONS_PER_REQUEST` distinct sessions; either breach answers 413 before the gate, and the same body is refused again on resend, so a sender splits its spool. Evidence: `src/codemie/repository/cli_analytics/postgres/session_binding.py::check_session_count`
- R9. Send `HEADER_CODEMIE_SESSION_ID` on an evidence request (one carrying a header of `EVIDENCE_HEADERS`) and keep the request to that one session: a missing id answers 400 `session_id_required` before anything is decoded, and a row of another session, or a row without one, discards the request as `evidence_session_conflict` (`check_boundary`). Evidence: `src/codemie/rest_api/routers/cli_analytics_ingest_route.py::IngestRoute`
- R10. Send the organization in `HEADER_CODEMIE_ORGANIZATION` or `HEADER_CODEMIE_ORGANIZATION_ENCODED`, not both; both headers, a malformed encoded value, or a `personal` or `corporate` claim without an organization on a request not routed via CodeMie answer 400 before classification. Evidence: `src/codemie/rest_api/routers/cli_analytics_ingest_route.py::_organization`
- R11. Treat an evidence request as stale only when it states both its subscription type and its policy version, the version differs from the server's and the stated type differs from the derived one; the ingestor then answers 409 `stale_policy` with the current `policy_version` after the session boundary check and before the gate (`_raise_if_stale`). Evidence: `src/codemie/rest_api/routers/cli_analytics_ingest_route.py::IngestRoute`
- R12. Check the publication-contract headers before the body is read: a malformed contract, or one without a session id, answers 422 with `disposition` `final`; a contract while `CLI_ANALYTICS_PUBLICATION_CONTRACTS_ENABLED` is off answers 503 with `disposition` `hold`. Evidence: `src/codemie/rest_api/routers/cli_analytics_ingest_route.py::check_publication_contract`

## Notes

The six events CodeMie CLI (formerly the `sdlc-analytics` plugin) sends on top of OTel telemetry and the 13 activity hook events (`src/codemie/repository/cli_analytics/vocabulary.py::DIMENSION_HOOK_TYPES`) (of these, the `lines_added` / `lines_removed` of `agent.tool.end`): their JSON, where each field is stored, and the rules a sender must keep. The storage, the recalculation and the operations are in [`cli-analytics-storage.md`](cli-analytics-storage.md).
The server accepts these events. A change to a field name, a type or a rule on this page is a change to the contract: agree it before a sender or the server implements it.

## Examples

### An ingest route: pre-body checks, gate, ingestor and error mapping

Source: `src/codemie/rest_api/routers/cli_analytics.py::ingest_event_hooks`

```python
async def ingest_event_hooks(request: Request) -> Response:
    _ensure_enabled()
    try:
        contract = check_publication_contract(request, request.state.user)
    except PublicationContractRejected as exc:
        return exc.response()
    ct = request.headers.get("content-type", "")
    if ct and not ct.startswith("application/x-ndjson"):
        raise HTTPException(status_code=415, detail="Expected application/x-ndjson")
    raw = await request.body()
    if len(raw) > config.ANALYTICS_INGEST_MAX_BODY_BYTES:
        raise HTTPException(status_code=413, detail=_ERR_BODY_TOO_LARGE)
    if not raw.strip():
        raise HTTPException(status_code=400, detail="Empty body")
    events = _parse_ndjson(raw)
    if not events:
        raise HTTPException(status_code=400, detail="No parseable events")
    now_ns = datetime_to_ns(datetime.now(timezone.utc))
    route = _ingest_route(request, contract)
    try:
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| An evidence request without a session id lost its range from the CLI spool | it was answered with a 200 discard, which the CLI acknowledges | answer 400 `session_id_required` before any decode or write | `src/codemie/rest_api/routers/cli_analytics_ingest_route.py::IngestRoute` |
| An NDJSON line is not stored and nothing reports it | `_parse_ndjson` skips a line that is not valid JSON, is not an object, or holds an integer too long to convert; a body with no line left is 400 | send one JSON object per line | `src/codemie/rest_api/routers/cli_analytics.py::_parse_ndjson` |
| A whole evidence request is discarded with `evidence_session_conflict` | a row carries a session other than the declared one, or no session | send one session per evidence request | `src/codemie/repository/cli_analytics/postgres/session_binding.py::check_boundary` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a hook field with its own column | add its name to `HOOK_TEXT_FIELDS` in column order with the column (storage guide); without that it stays in the residual attributes | `src/codemie/repository/cli_analytics/vocabulary.py::HOOK_TEXT_FIELDS` |
| Add an ingest failure status | subclass `TelemetryIngestError` and map it in `_INGEST_ERROR_STATUS`; an unmapped subclass answers 503 | `src/codemie/rest_api/routers/cli_analytics.py::_ingest_http_error` |
| Add an evidence header | add its constant to `EVIDENCE_HEADERS`; its presence makes a request an evidence request | `src/codemie/rest_api/routers/cli_analytics_ingest_route.py::EVIDENCE_HEADERS` |
| Advertise a capability to the CLI | add a key to the policy answer outside `version`, as `organization_encodings` is | `src/codemie/rest_api/routers/cli_analytics.py::classification_policy` |