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
- R3. Change date-window membership in `_admitted_arms` and its rule form `_SESSION_ADMITTED_IN_WINDOW` together; `_sessions_cte` builds `sel` from those arms and stays the only place the window is decided: a session is in a window when it started in it and has any activity, or has activity in the window though it started outside it, and the stored facts `has_activity`, `last_activity_day`, `resumed` stand for the rollup-table probes (NULL falls back to the tables). Evidence: `src/codemie/repository/cli_analytics/postgres/reader.py::_admitted_arms`
- R4. Change analytics tables with a revision in `src/external/alembic_cli_analytics/`; raw `usage_requests` is partitioned by `ts`. Evidence: `src/codemie/repository/cli_analytics/postgres/migrations.py::run_migrations`
- R5. Price CLI analytics money at read time from token counts and the platform catalog in the handler; a model without a price contributes 0 and is reported in `unpriced_models`. Evidence: `src/codemie/service/analytics/handlers/cli_analytics_handler.py::unpriced_models`
- R6. Expect schema migration at startup in the background and rollup and maintenance jobs on one pod at a time under PostgreSQL advisory locks. Evidence: `src/codemie/repository/cli_analytics/postgres/engine.py::pg_try_advisory_lock`
- R7. Run SQL changes against a real PostgreSQL: point `TEST_LOCAL_PG_URL` at it and run `poetry run pytest -n0 -rs tests/codemie/repository/cli_analytics/postgres/test_live_*.py`; a live test skips when the database is unreachable, so a skip is not a pass; unit tests need no database. Evidence: `tests/codemie/repository/cli_analytics/support/live_engine.py::get_live_engine`
- R8. Change a priced token kind or rate in `CatalogPricer.price` and in `cost_key_sql` (`_KIND_TOKENS`, `_RATE_COLUMNS`) together: the `cost_usd` session sort orders by the SQL cost key, fed `ModelRates` the service resolves with `model_rates_for` (the repository never imports the catalog). Evidence: `src/codemie/repository/cli_analytics/postgres/cost_key_sql.py::session_cost_key_sql`
- R9. Resolve the agent an Agent tool call started through `agent_call_attribution` (`agent_call_of`, `agent_call_agent_id`, `agent_call_ids_cte` for a session-wide read, `AGENT_CALL_TOOLS` for the tool names) rather than re-deriving it, so rollups, ingest marking and the reader agree. Evidence: `src/codemie/repository/cli_analytics/postgres/agent_call_attribution.py::agent_call_of`
- R10. Write each analytics revision to be applied alone: `run_migrations` upgrades one revision per connection and transaction under a session-level advisory lock, so a failure rolls back only that revision; to keep a cheap step (nullable columns) before a slow one that may hit the lock timeout, end it with `op.get_bind().commit()`. Evidence: `src/codemie/repository/cli_analytics/postgres/migrations.py::_upgrade_to`
- R11. Keep the Sessions list (`get_sessions`) reads inside `async with self._repo.bounded() as repo`, take a second connection only with `repo.widened()` around the statements that run concurrently, and price and build rows after the block (a request fanning out on the plain reader can deadlock a nearly exhausted pool shared with ingest; the other endpoints still `asyncio.gather` on the plain reader). Evidence: `src/codemie/repository/cli_analytics/postgres/reader.py::PostgresCliAnalyticsReader.bounded`
- R12. Attach the funding and personal-billed estimate with `buckets(price_bucket_rows(<get_session_funding_costs rows>), ...)`; never price those rows with the spend `CatalogPricer` the handler returns (`price_bucket_rows` builds its own, and each bucket reports its own `cost_incomplete` and `unpriced_models`). Evidence: `src/codemie/service/analytics/handlers/funding_buckets.py::price_bucket_rows`

### Constraints
| Do not | Do instead | Why | Evidence |
|---|---|---|---|
| Write `session_dims.started_at` or a day row of `turns_daily`, `tool_facts_daily`, `cost_daily`, `session_usage`, `lines_daily` outside `RollupRefresher._recompute` without deriving the facts | Call `derive_session_activity(conn, session_ids, timeout)` for those sessions in the same transaction | the Sessions window rule reads the stored `has_activity`/`last_activity_day`/`resumed` | `src/codemie/repository/cli_analytics/postgres/session_activity.py::derive_session_activity` |

## Notes

OTel telemetry from coding agents, sent by CodeMie CLI (formerly the sdlc-analytics plugin): Claude Code OTLP plus hook events, is stored in PostgreSQL, the only storage engine, behind the ports of `codemie.repository.cli_analytics`. The dashboards are served under `/v1/analytics/cli-analytics/*`.
OTel stays a data source, decoded inside the API: the OTLP endpoints of the API ingest directly, and a background job refreshes the rollups. There is no Collector and no second database; the analytics tables live in the application's PostgreSQL by default (see PostgreSQL Settings). Local stack: `docker compose up -d codemie postgres`.
