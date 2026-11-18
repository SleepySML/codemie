# Layered Architecture

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Router, service and repository layers and where startup wiring lives; package boundaries are in `architecture/project-structure.md`.

## Overview

The backend is a FastAPI app assembled in one module: it registers routers, middleware and exception handlers, and runs migrations and schedulers from its lifespan. Routers delegate to services under `src/codemie/service/`, and services reach PostgreSQL through SQLModel models and repositories or Elasticsearch through repository classes.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/rest_api/` | app assembly, routers, security dependencies | `src/codemie/rest_api/main.py::lifespan` |
| `src/codemie/service/` | business services | `src/codemie/service/assistant_service.py::AssistantService` |
| `src/codemie/repository/` | storage access for Elasticsearch, files and SQL aggregates | `src/codemie/repository/base_elastic_repository.py::BaseElasticRepository` |
| `src/codemie/core/` | shared exceptions, models, thought queue | `src/codemie/core/exceptions.py::ExtendedHTTPException` |
| `src/codemie/configs/` | settings object and logging setup | `src/codemie/configs/config.py::Config` |

## Layers

### Rules
- R1. Keep HTTP concerns in routers, orchestration in `src/codemie/service/`, and storage access in repositories or SQL models; repositories return data objects, not HTTP responses. Evidence: `src/codemie/service/assistant_service.py::AssistantService`
- R2. Gate an optional router where the app is assembled in `main.py` (as `ENABLE_USER_MANAGEMENT` does), not inside the router module. Evidence: `src/codemie/rest_api/main.py::ENABLE_USER_MANAGEMENT`
- R3. Use the shared exceptions in `codemie.core.exceptions` and settings from `config` instead of feature-local exception types or direct environment reads. Evidence: `src/codemie/core/exceptions.py::ValidationException`

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A new user-management or provisioning route returns 404 | user-management routers are included only when `ENABLE_USER_MANAGEMENT` is on, and `mount_provisioning_api` mounts `/v1/provisioning` only when `PROVISIONING_API_ENABLED` and `ENABLE_USER_MANAGEMENT` are both on | enable the flags in the environment, or include the router outside the flag block when it is unconditional | `src/codemie/rest_api/routers/provisioning/__init__.py::mount_provisioning_api` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Register a new router | import its module and add an `app.include_router` line in the registration area | `src/codemie/rest_api/main.py::include_router` |
| Add a startup step | add a helper and call it from the lifespan | `src/codemie/rest_api/main.py::lifespan` |
| Gate a router behind a feature flag | wrap its include in the config check where the app is assembled | `src/codemie/rest_api/main.py::register_marketplace_management_router` |
