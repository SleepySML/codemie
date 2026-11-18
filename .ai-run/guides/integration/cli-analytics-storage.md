# CLI Analytics Storage

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: PostgreSQL storage of CLI analytics: settings, windows, schema, money and operations; the event contract is in `integration/cli-analytics-events.md`.

## Overview

CLI analytics storage is built once on first use: `get_cli_analytics_storage` creates one PostgreSQL engine shared by the reader, the ingestor and the runtime jobs.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/repository/cli_analytics/factory.py` | storage construction | `src/codemie/repository/cli_analytics/factory.py::get_cli_analytics_storage` |
| `src/codemie/repository/cli_analytics/postgres/` | reader, ingestor, rollups, runtime | `src/codemie/repository/cli_analytics/factory.py::_build_storage` |
| `src/external/alembic_cli_analytics/` | analytics schema migrations | `src/external/alembic_cli_analytics/env.py::context.configure` |

## CLI analytics storage

### Rules
- R1. Read and write CLI analytics through the ports (`CliAnalyticsReader`, `CliTelemetryIngestor`, `CliAnalyticsRuntime`) of the storage `get_cli_analytics_storage` builds; it builds one engine shared by reader, ingestor and jobs. Evidence: `src/codemie/repository/cli_analytics/factory.py::_build_storage`
- R2. Configure a separate analytics database with `CLI_ANALYTICS_PG_URL` and the schema with `CLI_ANALYTICS_PG_SCHEMA` (default `codemie_analytics`); empty URL means the application database. Evidence: `src/codemie/configs/config.py::CLI_ANALYTICS_PG_SCHEMA`
- R3. Change date-window membership in `_sessions_cte` alone: a session is in a window when `session_dims.started_at` falls inside it. Evidence: `src/codemie/repository/cli_analytics/postgres/reader.py::_sessions_cte`
- R4. Change analytics tables with a revision in `src/external/alembic_cli_analytics/`; raw `usage_requests` is partitioned by `ts`. Evidence: `src/codemie/repository/cli_analytics/postgres/migrations.py::run_migrations`
- R5. Price CLI analytics money at read time from token counts and the platform catalog in the handler; a model without a price contributes 0 and is reported in `unpriced_models`. Evidence: `src/codemie/service/analytics/handlers/cli_analytics_handler.py::unpriced_models`
- R6. Expect schema migration at startup in the background and rollup and maintenance jobs on one pod at a time under PostgreSQL advisory locks. Evidence: `src/codemie/repository/cli_analytics/postgres/engine.py::pg_try_advisory_lock`
- R7. Run SQL changes against real PostgreSQL with `make test-cli-analytics-live`; unit tests need no database. Evidence: `Makefile::test-cli-analytics-live`

## Notes

OTel telemetry from coding agents, sent by CodeMie CLI (formerly the sdlc-analytics plugin): Claude Code OTLP plus hook events, is stored in PostgreSQL, the only storage engine, behind the ports of `codemie.repository.cli_analytics`. The dashboards are served under `/v1/analytics/cli-analytics/*`.
OTel stays a data source, decoded inside the API: the OTLP endpoints of the API ingest directly, and a background job refreshes the rollups. There is no Collector and no second database; the analytics tables live in the application's PostgreSQL by default (see PostgreSQL Settings). Local stack: `docker compose up -d codemie postgres`.
