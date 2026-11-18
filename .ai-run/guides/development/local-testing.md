# Local Testing

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Test scope and environment blocks when running checks locally; test authoring is in `testing/testing-patterns.md`.

## Overview

`make test` runs pytest over `tests/` with `pytest.ini` pinning `ENV=local` and an unreachable `PG_URL`, and a session fixture replaces the PostgreSQL engine with a mock. Enterprise tests run by default and are skipped with `--skip-enterprise`.

| Part | What it holds | Start here |
|---|---|---|
| `pytest.ini` | paths, env pins, markers | `pytest.ini::markers` |
| `tests/conftest.py` | session fixtures and the enterprise skip option | `tests/conftest.py::mock_database_engine` |

## Local checks

### Rules
- R1. Run tests only when asked, the narrowest scope first, and report a gate that did not run as not run. Evidence: `Makefile::test`
- R2. Report a missing credential, service or enterprise package as a prerequisite, not a code failure; `--skip-enterprise` skips enterprise-marked tests. Evidence: `tests/conftest.py::pytest_collection_modifyitems`

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| The suite fails at the start with a missing-plugin assertion | `require_pytest_plugins` fails the session when a required pytest plugin is not importable | run through `poetry run pytest` after `make install` | `tests/conftest.py::require_pytest_plugins` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Run one test file | `poetry run pytest <path>` | `Makefile::test` |
| Skip enterprise tests | add `--skip-enterprise` | `tests/conftest.py::pytest_addoption` |
| Run the live-stack tests | set `RUN_LIVE_STACK_TESTS=1` and select `-m live_stack` | `tests/codemie/service/analytics/handlers/test_workflow_executions_live.py::pytestmark` |
