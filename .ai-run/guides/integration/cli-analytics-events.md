# CLI Analytics Events

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Contract of the CLI plugin events on the ingest routes; storage, windows and money are in `integration/cli-analytics-storage.md`.

## CLI analytics events

### Rules
- R1. Accept CLI events on `/v1/analytics/cli-analytics/event-hooks` (NDJSON) and the OTLP `logs`, `metrics` and `traces` routes; the sender comes from the token. Evidence: `src/codemie/rest_api/routers/cli_analytics.py::ingest_event_hooks`
- R2. Decide store or discard through the `IngestGate` before anything is written; a discarded body answers 200, because the CLI deletes its spool on any 2xx. Evidence: `src/codemie/repository/cli_analytics/ports.py::IngestGate`
- R3. Serve the corporate organization list from `classification-policy` and classify sessions on the server from the routing and organization headers; the organization name is not stored. Evidence: `src/codemie/rest_api/routers/cli_analytics.py::classification_policy`
- R4. Send one flat snake_case JSON object per line whose column fields carry the column's name, with the envelope `type`, `timestamp`, `session_id`, `prompt_id`. Evidence: `src/codemie/repository/cli_analytics/postgres/ingestor.py::lines_added`
- R5. Deduplicate an event with an `event_id` on `(type, session_id, event_id)`; one without it is identified by its whole content. Evidence: `src/codemie/repository/cli_analytics/postgres/otlp.py::event_id`
- R6. Show a private Personal session to its owner alone; `personal_only` filters the sessions list. Evidence: `src/codemie/rest_api/routers/cli_analytics.py::personal_only`
- R7. Split the event contract so each guide stays under the 400-line cap.

## Notes

The six events CodeMie CLI (formerly the `sdlc-analytics` plugin) sends on top of OTel telemetry and the 13 activity hook events (of these, the `lines_added` / `lines_removed` of `agent.tool.end`): their JSON, where each field is stored, and the rules a sender must keep. The storage, the recalculation and the operations are in [`cli-analytics-storage.md`](cli-analytics-storage.md).
The server accepts these events. A change to a field name, a type or a rule on this page is a change to the contract: agree it before a sender or the server implements it.
