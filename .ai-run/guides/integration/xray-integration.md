# X-ray Integration

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: X-ray datasource and QA tools; Jira is in `integration/jira-integration.md`.

## Overview

X-ray tests are indexed by `XrayDatasourceProcessor`; agents reach X-ray through the QA toolkit's get, create and GraphQL tools.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/datasource/xray/` | X-ray datasource processor | `src/codemie/datasource/xray/xray_datasource_processor.py::XrayDatasourceProcessor` |
| `src/codemie_tools/qa/xray/` | X-ray tools | `src/codemie_tools/qa/xray/tools.py::XrayGetTestsTool` |
| `src/codemie_tools/qa/toolkit.py` | QA toolkit (X-ray, Zephyr) | `src/codemie_tools/qa/toolkit.py::QualityAssuranceToolkit` |

## X-ray

### Rules
- R1. Keep X-ray behaviour in `XrayDatasourceProcessor` and the QA toolkit's X-ray tools, apart from Jira code. Evidence: `src/codemie_tools/qa/toolkit.py::QualityAssuranceToolkit`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Change test retrieval | edit `XrayGetTestsTool` | `src/codemie_tools/qa/xray/tools.py::XrayGetTestsTool` |
| Change test creation | edit `XrayCreateTestTool` | `src/codemie_tools/qa/xray/tools.py::XrayCreateTestTool` |
| Change X-ray indexing | edit the processor's `_init_loader` | `src/codemie/datasource/xray/xray_datasource_processor.py::XrayDatasourceProcessor._init_loader` |
