# Setup Guide

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Environment files, local dependencies, running the API and the Claude Code Stop hook; gate commands are in `quality-gates.md`.

## Overview

Local runs need Poetry dependencies, a `.env` copied from `.env.example`, and the Docker Compose services; `make run` starts Uvicorn with reload on port 8080, and app startup applies Alembic migrations.

| Part | What it holds | Start here |
|---|---|---|
| `Makefile` | install, run and gate targets | `Makefile::run` |
| `src/codemie/configs/config.py` | env file loading | `src/codemie/configs/config.py::load_env_files` |
| `docker-compose.yml` | local service stack | `README.md` § Running the Application |

## Local setup

### Rules
- R1. Copy `.env.example` to `.env` and put machine-specific overrides in `.env.local`; both are gitignored and `.env.local` wins over `.env`. Evidence: `src/codemie/configs/config.py::load_env_files`
- R2. Install with `make install` and start PostgreSQL and Elasticsearch with Docker Compose before the app; startup applies migrations. Evidence: `src/codemie/rest_api/main.py::_initialize_database_and_defaults`
- R3. Start the API with `make run` (Uvicorn on `codemie.rest_api.main:app`, port 8080). Evidence: `Makefile::run`
- R4. Put Poetry on the login-shell PATH (`~/.profile`, `~/.zprofile` or `/etc/paths.d/poetry` on macOS): the Stop hook runs `make ruff` in a non-login shell, and `env -i PATH="$PATH" poetry --version` checks it. Evidence: `Makefile::ruff`

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| The Claude Code Stop hook fails with `poetry: No such file or directory` | the hook runs `make ruff` in a non-login shell without the profile PATH | add Poetry to the login profile, see the guide's Stop hook section | `Makefile::ruff` |
| Startup fails on a database error | startup runs `alembic_upgrade_postgres` against `PG_URL` before serving | start PostgreSQL from Docker Compose first | `src/codemie/rest_api/main.py::_initialize_database_and_defaults` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Install dependencies | `make install` (or `make install-enterprise` with the extra) | `Makefile::install` |
| Install the git hooks | `make install-hooks` | `Makefile::install-hooks` |
| Start the API | `make run` | `Makefile::run` |
