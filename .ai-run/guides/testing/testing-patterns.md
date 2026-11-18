# Testing Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: pytest layout, test shape, fixtures, mocking, database and API tests, coverage and test environment; router and service specifics are in `testing/testing-api-patterns.md` and `testing/testing-service-patterns.md`.

## Overview

pytest collects `tests/` with `pythonpath = src`, xdist (`-n 3`) and env pins from `pytest.ini`; a session fixture replaces the PostgreSQL engine with a `MagicMock`, so unit tests touch no database. Tests mirror the source tree under `tests/codemie/` and `tests/codemie_tools/`.

| Part | What it holds | Start here |
|---|---|---|
| `pytest.ini` | collection, env pins, markers | `pytest.ini::addopts` |
| `tests/conftest.py` | session fixtures | `tests/conftest.py::mock_database_engine` |
| `tests/codemie/` | tests of the backend package | `tests/codemie/clients/test_postgres_transactional_session.py::TestTransactionalSession` |
| `tests/codemie_tools/` | tests of the toolkits | `tests/codemie_tools/base/test_toolkit_provider.py::test_integration_all_tool_configs_produce_info` |

## Organization and naming

### Rules
- R1. Mirror `src/` under an existing test root and reuse its fixtures instead of a new top-level test tree; mock provider boundaries unless the test is an integration test. Evidence: `pytest.ini::testpaths`
- R2. Put a test under `tests/codemie/` or `tests/codemie_tools/` at the path that mirrors its source module, named `test_<module>.py` (pytest collects `testpaths = tests` with `pythonpath = src`). Evidence: `pytest.ini::testpaths`
- R3. Mark a test that needs the enterprise package `enterprise` (the only marker declared in `pytest.ini`; `--skip-enterprise` skips it); give a test that needs the running docker stack a module `pytestmark` of `pytest.mark.live_stack` plus `skipif(os.getenv("RUN_LIVE_STACK_TESTS") != "1")`, since the undeclared marker gates nothing by itself. Evidence: `tests/codemie/service/analytics/handlers/test_workflow_executions_live.py::pytestmark`

## Running tests

### Rules
- R1. While iterating, run one file with `poetry run pytest <path>` and one case with `-k <name>`; leave the full suite to the test gate in `.ai-run/guides/quality-gates.md` and state the exact command and scope. Evidence: `Makefile::test`

## Test shape

### Rules
- R1. Group the cases of one unit in a `Test<Unit>` class and name each case for the behaviour it asserts, such as `test_returns_empty_when_no_matching_bucket`. Evidence: `tests/codemie/service/stale_datasource/test_stale_datasource_service.py::TestFetchLifecycleMetrics`
- R2. Mark an async case `@pytest.mark.asyncio` and await the call (`pytest.ini` sets the fixture loop scope per function). Evidence: `pytest.ini::asyncio_default_fixture_loop_scope`
- R3. Write table cases with `@pytest.mark.parametrize` rather than copied test bodies. Evidence: `tests/codemie/rest_api/a2a/test_types.py::test_response_fixture_round_trips`

## Fixtures and factories

### Rules
- R1. Share fixtures through the nearest `conftest.py`; session-wide setup belongs in `tests/conftest.py`, which already replaces `PostgresClient.get_engine` with a `MagicMock` for the session. Evidence: `tests/conftest.py::mock_database_engine`
- R2. Undo what a test changes in fixture teardown, with `monkeypatch` or a `yield` fixture (process state a test leaves behind changes what a later test sees). Evidence: `tests/conftest.py::isolate_run_reporting_state`

## Mocking

### Rules
- R1. For a helper that encodes a decision, add one callsite test per branch asserting the value that reaches the boundary (SDK call, request), not only a unit test of the helper. Evidence: `tests/codemie/rest_api/routers/test_projects_router.py::TestProjectCreationEndpoint.test_create_project_calls_service_directly`
- R2. Patch a name in the module that looks it up, such as `codemie.rest_api.routers.projects.crud.project_service`, with `unittest.mock.patch`; use `patch.object` for attributes of a class or of `config`. Evidence: `tests/codemie/rest_api/routers/test_projects_router.py::TestProjectCreationEndpoint.test_create_project_calls_service_directly`
- R3. Mock external HTTP, provider SDKs and the database in unit tests; reach real services only in tests marked `live_stack` or gated on `TEST_LOCAL_PG_URL` or `HOSTED_TEST_DATABASE_URL`. Evidence: `tests/codemie/repository/cli_analytics/postgres/test_live_route_pricing.py::pytestmark`

## Database testing

### Rules
- R1. Test transaction behaviour on an in-memory SQLite engine patched into `PostgresClient.get_engine`, creating the tables the test needs. Evidence: `tests/codemie/clients/test_postgres_transactional_session.py::sqlite_engine`
- R2. Put PostgreSQL-specific SQL tests of CLI analytics in `test_live_*` modules under `tests/codemie/repository/cli_analytics/postgres/`, taking the engine from `get_live_engine` (it skips when no PostgreSQL is reachable at `TEST_LOCAL_PG_URL`, default `localhost:5432`); run them with `TEST_LOCAL_PG_URL=... poetry run pytest <dir> -k live -rs` and treat a skip as not run. Service-level PostgreSQL tests live in `*_pg.py` modules whose `pytestmark` skips without `TEST_LOCAL_PG_URL` or `HOSTED_TEST_DATABASE_URL`. Evidence: `tests/codemie/repository/cli_analytics/support/live_engine.py::get_live_engine`
- R3. Test hosted-applications and permission SQL that needs PostgreSQL by requesting the `database` fixture from `tests/codemie/service/hosted_applications/conftest.py` rather than building a new engine; it skips unless `HOSTED_TEST_DATABASE_URL` points at an isolated PostgreSQL database, installs the hosted migrations plus `users`/`user_projects` in a fresh `hosted_test_<hex>` schema, and drops it on teardown. Evidence: `tests/codemie/service/hosted_applications/conftest.py::database`
- R4. When a SQLite test depends on foreign-key enforcement or SAVEPOINTs (`begin_nested`), build the engine with `constrained_sqlite_engine()` from `tests/codemie/sqlite_harness.py` instead of a bare `create_engine("sqlite://")` (SQLite leaves foreign keys off and pysqlite's own BEGIN breaks SAVEPOINT, so the test passes or fails for the wrong reason); it also compiles JSONB and ARRAY as JSON. Evidence: `tests/codemie/sqlite_harness.py::constrained_sqlite_engine`

### Constraints
| Do not | Do instead | Why | Evidence |
|---|---|---|---|
| Do not pin a rewritten or batched query only by its SQL text or against a literal expected value | Do instead seed the same data and assert the new read returns what the read it replaces returns, on Postgres in a `test_live_*` module for SQL, as `TestEveryFilterOfABatchGetsTheCardsItsOwnReadGives` does | a SQL-text assertion is built with the builder's own escaping and a literal does not move when the original changes, so neither catches a divergence | `tests/codemie/service/spend/test_spend_group_aggregation.py::TestEveryFilterOfABatchGetsTheCardsItsOwnReadGives` |

## API testing

### Rules
- R1. For an HTTP-level test, build `TestClient(app)` in a fixture, override `authenticate` with a `User`, and clear `app.dependency_overrides` afterwards. Evidence: `tests/codemie/rest_api/routers/test_conversation_metrics_access.py::client`

## Coverage

### Rules
- R1. Read coverage for `src/codemie` and `src/codemie_tools`: `.coveragerc` includes those two trees, omits templates and the provider client, and sets no fail-under threshold. Evidence: `.coveragerc::include`

## Test environment

### Rules
- R1. Rely on the `pytest.ini` env pins (`ENV=local`, a dummy `PG_URL`, `MARKETPLACE_APPROVAL_GATE_ENABLED=false`) rather than your `.env`; a test that needs the approval gate on patches `config.MARKETPLACE_APPROVAL_GATE_ENABLED` itself. Evidence: `pytest.ini::MARKETPLACE_APPROVAL_GATE_ENABLED`

## Examples

### A session fixture that blocks real database connections

Source: `tests/conftest.py::mock_database_engine`

```python
def mock_database_engine():
    """
    Mock database engine to prevent actual database connections during tests.

    This fixture is automatically applied to all tests (autouse=True) and runs once
    per test session (scope="session"). It ensures that no actual database connections
    are made during test execution, even when modules are imported.
    """
    with patch("codemie.clients.postgres.PostgresClient.get_engine") as mock_get_engine:
        # Return a mock engine that won't try to connect to a real database
        mock_engine = MagicMock()
        mock_engine.__enter__ = MagicMock(return_value=mock_engine)
        mock_engine.__exit__ = MagicMock(return_value=False)
        mock_get_engine.return_value = mock_engine
        yield mock_engine
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A log or billing assertion sees an extra metric line or a project the test did not pass | importing `codemie.rest_api.main` registers metric sinks for the process, and a `litellm_context` a test sets outlives it | rely on the autouse fixture that clears the sinks and resets the context; a test that needs a sink adds it itself | `tests/conftest.py::isolate_run_reporting_state` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Run one file | `poetry run pytest <file>` | `pytest.ini::addopts` |
| Run one case | add `-k <name>` | `pytest.ini::addopts` |
| Skip enterprise tests | pass `--skip-enterprise` | `tests/conftest.py::pytest_addoption` |
