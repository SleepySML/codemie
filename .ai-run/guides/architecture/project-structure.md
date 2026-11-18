# Project Structure

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Package boundaries of `codemie` and `codemie_tools`; layer responsibilities are in `architecture/layered-architecture.md`.

## Overview

Two packages ship from `src`: `codemie` (the backend) and `codemie_tools` (tool integrations agents call). `src/external/` holds Alembic trees and deployment scripts that Ruff ignores.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/` | backend application | `src/codemie/rest_api/main.py::app` |
| `src/codemie_tools/` | toolkits and tools | `src/codemie_tools/base/codemie_tool.py::CodeMieTool` |
| `src/external/alembic/` | PostgreSQL migrations | `src/codemie/clients/postgres.py::alembic_upgrade_postgres` |
| `src/external/deployment_scripts/` | preconfigured assistants, skills and workflows | `src/external/deployment_scripts/preconfigured_assistants.py::manage_preconfigured_assistants` |

## Packages

### Rules
- R1. Put backend code in `src/codemie/` (rest_api, service, repository, workflows and the agent runtime) and agent toolkits in `src/codemie_tools/`. Evidence: `pyproject.toml::packages`
- R2. Put workflow node behaviour in `src/codemie/workflows/nodes/` and provider logic in services or toolkits; extend an existing package before adding a top-level one. Evidence: `src/codemie/workflows/nodes/base_node.py::BaseNode`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a toolkit | create a package under `src/codemie_tools/` with toolkit, models and tools modules | `src/codemie_tools/notification/toolkit.py::NotificationToolkit` |
| Add a workflow node | add a module under `src/codemie/workflows/nodes/` | `src/codemie/workflows/nodes/base_node.py::BaseNode` |
| Add a migration | add a revision under `src/external/alembic/versions/` | `src/codemie/clients/postgres.py::alembic_upgrade_postgres` |
