# Jira Integration

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Jira datasource and project-management tools; Confluence is in `integration/confluence-integration.md`.

## Overview

Jira issues are indexed by `JiraDatasourceProcessor`, which builds a `JiraLoader` from the JQL, custom fields and credentials; agent-side Jira tools sit under `src/codemie_tools/core/project_management/`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/datasource/jira/` | Jira datasource processor | `src/codemie/datasource/jira/jira_datasource_processor.py::JiraDatasourceProcessor` |
| `src/codemie_tools/core/project_management/` | Jira and Confluence tools | `src/codemie_tools/core/project_management/xwiki/toolkit.py::XWikiToolkit` |

## Jira

### Rules
- R1. Change Jira indexing in `JiraDatasourceProcessor` and agent Jira calls in `codemie_tools/core/project_management`, not in routers. Evidence: `src/codemie/datasource/jira/jira_datasource_processor.py::JiraDatasourceProcessor._init_loader`

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| An incremental Jira reindex re-reads the whole project | the loader filters by `last_reindex_date` only when the run is an incremental reindex and the index has that date | check `is_incremental_reindex` and `last_reindex_date` before suspecting the loader | `src/codemie/datasource/jira/jira_datasource_processor.py::JiraDatasourceProcessor._init_loader` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Change which issues are indexed | adjust the JQL passed to the processor | `src/codemie/datasource/jira/jira_datasource_processor.py::JiraDatasourceProcessor._init_loader` |
| Index extra custom fields | pass `custom_fields`; cron runs fall back to the stored index config | `src/codemie/datasource/jira/jira_datasource_processor.py::JiraDatasourceProcessor._effective_custom_fields` |
| Change the Jira index name | edit `_index_name` | `src/codemie/datasource/jira/jira_datasource_processor.py::JiraDatasourceProcessor._index_name` |
