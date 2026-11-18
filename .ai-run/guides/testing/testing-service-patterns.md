# Testing Service Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Service test isolation and async service tests; suite-wide test patterns are in `testing/testing-patterns.md`.

## Overview

Service tests build the service with its collaborators mocked and assert decisions and calls; async service methods are tested with `@pytest.mark.asyncio`, and the fixture loop scope is per function.

| Part | What it holds | Start here |
|---|---|---|
| `tests/codemie/service/` | service tests, mirroring `src/codemie/service/` | `tests/codemie/service/stale_datasource/test_stale_datasource_service.py::TestFetchLifecycleMetrics` |

## Service tests

### Rules
- R1. Test service decisions with repositories and providers mocked; hit live services only in marked integration tests. Evidence: `tests/codemie/service/stale_datasource/test_stale_datasource_service.py::TestComputeBaseToolNames`
- R2. Mark async service tests `@pytest.mark.asyncio` and await the call. Evidence: `tests/codemie/service/stale_datasource/test_stale_datasource_service.py::TestFetchLifecycleMetrics`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Test an async service method | mark the test `@pytest.mark.asyncio` and await the call | `tests/codemie/service/stale_datasource/test_stale_datasource_service.py::TestFetchLifecycleMetrics.test_returns_last_seen_for_matching_datasource` |
| Test a transaction against a real engine | patch `PostgresClient.get_engine` to an in-memory SQLite engine | `tests/codemie/clients/test_postgres_transactional_session.py::sqlite_engine` |
| Find the tests of a service | open the mirrored path under `tests/codemie/service/` | `tests/codemie/service/stale_datasource/test_stale_datasource_service.py::TestComputeBaseToolNames` |
