# Agent Tools

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Agent-facing tool ownership and schema compatibility; toolkit creation is in `agents/custom-tool-creation.md`.

## Overview

Agent-facing adapters live in `src/codemie/agents/tools/`; reusable integrations live in `src/codemie_tools/`. Tool JSON schemas are rewritten for Google Vertex AI by deep-copying and converting `integer` to `number` inside `anyOf`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/agents/tools/` | agent tool adapters (kb, code, platform, skill, ide) | `src/codemie/agents/tools/schema_compatibility.py::transform_schema_for_gcp_compatibility` |
| `src/codemie_tools/base/` | tool base class and registry | `src/codemie_tools/base/codemie_tool.py::CodeMieTool` |

## Agent tools

### Rules
- R1. Put reusable integrations in a `codemie_tools` toolkit and agent runtime adapters under `src/codemie/agents/tools/`; provider API calls stay out of prompts and routers. Evidence: `src/codemie/agents/tools/schema_compatibility.py::transform_schema_for_gcp_compatibility`
- R2. Pass a tool schema through `transform_schema_for_gcp_compatibility` before Vertex AI sees it and cover a schema change in the schema compatibility tests (Vertex rejects `integer` inside `anyOf`). Evidence: `src/codemie/agents/tools/schema_compatibility.py::transform_schema_for_gcp_compatibility`

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A tool schema works on OpenAI but Vertex AI rejects it | Vertex AI does not accept `integer` inside `anyOf` | run the schema through the GCP transform and cover it in its test module | `src/codemie/agents/tools/schema_compatibility.py::transform_schema_for_gcp_compatibility` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Make a schema Vertex-compatible | pass it through `transform_schema_for_gcp_compatibility` | `src/codemie/agents/tools/schema_compatibility.py::transform_schema_for_gcp_compatibility` |
| Sanitize a tool response for Gemini | use `sanitize_tool_response_for_gcp` | `src/codemie/agents/tools/schema_compatibility.py::sanitize_tool_response_for_gcp` |
| Test a schema change | extend the schema compatibility tests | `tests/codemie/agents/tools/test_schema_compatibility.py::TestTransformSchemaForGCPCompatibility` |
