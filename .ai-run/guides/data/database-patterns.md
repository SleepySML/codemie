# Database Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: SQLModel sessions, models, transactions and Alembic migrations; pagination and search limits are in `data/database-optimization.md`.

## Overview

PostgreSQL access goes through SQLModel models on `BaseModelWithSQLSupport` and the engine and session helpers of `PostgresClient`. `get_session` joins an active `transactional_session`, so `save()` and `update()` take part in one transaction; Alembic migrations run to head at app startup.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/clients/postgres.py` | engines, sessions, transactions, migration runner | `src/codemie/clients/postgres.py::PostgresClient` |
| `src/codemie/rest_api/models/base.py` | SQL and Elasticsearch model bases with CRUD helpers | `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport` |
| `src/codemie/repository/` | repositories for aggregates that need queries | `src/codemie/repository/budget_repository.py::BudgetRepository` |
| `src/external/alembic/` | migration environment and versions | `src/codemie/clients/postgres.py::alembic_upgrade_postgres` |

## Key abstractions

| Abstraction | What it is | Where |
|---|---|---|
| BaseModelWithSQLSupport | CRUD helpers: `save` assigns a UUID id and dates, `update` raises `StaleDataError` for a deleted row | `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport.update` |
| transactional_session | ambient session: one commit on success, rollback on exception, re-entrant | `src/codemie/clients/postgres.py::transactional_session` |

## Connection and configuration

### Rules
- R1. Get SQL sessions from `get_session` (sync) or `get_async_session` (async) in `src/codemie/clients/postgres.py` rather than building an engine (`PostgresClient` keeps one engine per process id). Evidence: `src/codemie/clients/postgres.py::PostgresClient.get_engine`
- R2. Read the spend ledger through `get_ledger_async_session`, which moves to its own pool and statement timeout when `SPEND_LEDGER_POOL_SIZE` is above 0. Evidence: `src/codemie/clients/postgres.py::PostgresClient.get_ledger_async_engine`

## Data access

### Rules
- R1. Define a table as a SQLModel class on `BaseModelWithSQLSupport` with `table=True`; it brings a string `id` primary key (a UUID4 on `save`), `date` and `update_date`. Evidence: `src/codemie/rest_api/models/base.py::CommonBaseModel`
- R2. Store a nested Pydantic value in a JSON column through `PydanticType` or `PydanticListType`. Evidence: `src/codemie/rest_api/models/base.py::PydanticType`

### Constraints
| Do not | Do instead | Why | Evidence |
|---|---|---|---|
| Do not add a column holding a `users.id` without a foreign key (plain `user_id`/`created_by`, or a `CreatedByUser`/`UserEntity` JSONB column) without registering it | add it to the matching `_PLAIN_ID_COLUMNS`, `_CREATED_BY_JSONB_COLUMNS` or `_USER_ENTITY_JSONB_COLUMNS` tuple in `user_remap_service.py`; only real FKs to `users.id` are found from `pg_constraint` | the user id remap iterates the fixed tuples, so an unlisted column keeps the old id and nothing signals it | `src/codemie/service/user/user_remap_service.py::UserRemapService._update_plain_id_columns` |

## Queries

### Rules
- R1. Filter with `get_by_fields` or `get_all_by_fields`; a dotted key such as `created_by.name` becomes a JSON path through `get_field_expression`. Evidence: `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport.get_field_expression`
- R2. Build a SQL substring search with `search_ranking(text, *columns)`, which escapes LIKE wildcards and returns `(condition, rank)`; filter by the condition and put the rank first in ORDER BY when a search term is given. Evidence: `src/codemie/core/search_ranking.py::search_ranking`

### Constraints
| Do not | Do instead | Why | Evidence |
|---|---|---|---|
| Do not compare an `sa.Enum` column with a literal string such as `"assistant"` | compare with the enum member (`Permission.resource_type == ResourceType.ASSISTANT`), as `Permission.get_for` does; the PostgreSQL labels are the member names (`ASSISTANT`), not the Python values | a lowercase literal matches no row, so the filter silently returns nothing | `src/codemie/rest_api/models/permission.py::Permission.get_for` |

## Transactions

### Rules
- R1. Wrap writes that must commit together in `transactional_session()`; `save()` and `update()` join it through `get_session`, and it rolls back when the block raises. Evidence: `src/codemie/clients/postgres.py::transactional_session`
- R2. Delete inside a `transactional_session` through the session the block yields; `BaseModelWithSQLSupport.delete()` opens its own session and commits at once. Evidence: `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport.delete`

### Constraints
| Do not | Do instead | Why | Evidence |
|---|---|---|---|
| Do not copy `src/codemie/service/project/project_assignment_service.py::ProjectAssignmentService.assign_user_to_project` (check-then-insert with no `IntegrityError` handling) | insert and map the unique violation to 409, as `src/codemie/service/cost_center_service.py::CostCenterService.create` does | two concurrent requests both pass the existence check and the loser answers 500 | `src/codemie/service/cost_center_service.py::CostCenterService.create` |

## Migrations and seeds

### Rules
- R1. Add a revision under `src/external/alembic/versions/` and check heads with `make check-alembic` (it merges in origin/main's versions; the pre-push hook and the MR pipeline run the same script) before merging, since startup upgrades to `head`, which fails with more than one head. Evidence: `Makefile::check-alembic`, `src/codemie/clients/postgres.py::alembic_upgrade_postgres`
- R2. Change a PostgreSQL enum with `op.sync_enum_values`, listing the full member set in upgrade and downgrade; to only add a value to an enum other branches also extend (e.g. `codemie.resourcetype`), run `ALTER TYPE ... ADD VALUE IF NOT EXISTS` inside `op.get_context().autocommit_block()` (a synced fixed list would drop values another branch added). Evidence: `src/external/alembic/versions/8b2c1a4d5e6f_add_ms_teams_credential_type.py::upgrade`, `src/external/alembic/versions/f8d24a6c0b17_platform_sharing_grants_and_links.py::upgrade`
- R3. Put CLI analytics schema changes in the separate tree `src/external/alembic_cli_analytics/`. Evidence: `src/external/alembic_cli_analytics/env.py::context.configure`

## Safety

### Rules
- R1. Build queries with SQLModel or SQLAlchemy expressions, not string interpolation of values. Evidence: `src/codemie/repository/scheduler_run_repository.py::SchedulerRunRepository.list_runs`
- R2. Bind values in raw SQL as `:name` parameters of `text()`; an f-string splices clauses and placeholders the code builds itself. Evidence: `src/codemie/repository/scheduler_run_repository.py::SchedulerRunRepository.list_runs`
- R3. Catch `KeyError` from `get_by_id` and `StaleDataError` from `update` in the service and raise a typed exception. Evidence: `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport.update`

## Examples

### Joining an ambient transaction

Source: `src/codemie/clients/postgres.py::get_session`

```python
def get_session(engine=None):
    """
    Context manager for database sessions.

    If a `transactional_session()` is active in the current context, joins it (as a
    `_JoinedSession` whose commit/rollback/close are no-ops) so the work participates in that
    single transaction. Otherwise opens and owns its own session, exactly as before.

    Args:
        engine: Engine for an owned session; defaults to `PostgresClient.get_engine()`.

    Yields:
        Session: SQLModel session for database operations.
    """
    ambient = _current_session.get()
    if ambient is not None:
        yield _JoinedSession(ambient)
        return

    with Session(engine or PostgresClient.get_engine()) as session:
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A delete inside `transactional_session` is committed even though the block later rolls back | `BaseModelWithSQLSupport.delete` opens its own `Session` and commits instead of using `get_session` | delete through the session the transaction yields | `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport.delete` |
| `get_by_id` raises `KeyError` and the API returns 500 | `get_by_id` raises `KeyError` for a missing row; `find_by_id` returns an empty result | use `find_by_id` and raise a typed 404 | `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport.get_by_id` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Read a row that may be absent | call `find_by_id` | `src/codemie/rest_api/models/base.py::BaseModelWithSQLSupport.find_by_id` |
| Write two rows atomically | open `transactional_session` and save both inside it | `src/codemie/clients/postgres.py::transactional_session` |
| Query from async code | use `get_async_session` | `src/codemie/clients/postgres.py::get_async_session` |
| Resolve two Alembic heads | merge them before startup, which upgrades to a single head | `src/codemie/clients/postgres.py::alembic_upgrade_postgres` |
