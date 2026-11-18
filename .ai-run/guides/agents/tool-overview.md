# Tool Overview

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Toolkit layers and the static tool registry; creating a toolkit is in `agents/custom-tool-creation.md`.

## Overview

Two tool layers exist: platform adapters in `src/codemie/agents/tools/` and reusable toolkits in `src/codemie_tools/`. `tool_registry` maps a tool name to the classes that carry it and their class flags; `ensure_loaded` imports the tool packages once and logs, rather than raises, a failed import.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/agents/tools/` | platform tool adapters | `src/codemie/agents/tools/schema_compatibility.py::transform_schema_for_gcp_compatibility` |
| `src/codemie_tools/` | reusable toolkits per external system | `src/codemie_tools/base/toolkit_provider.py::is_toolkit_class` |
| `src/codemie_tools/base/tool_registry.py` | static tool facts | `src/codemie_tools/base/tool_registry.py::ToolRecord` |

## Tool layers

### Rules
- R1. Place platform tool adapters in `src/codemie/agents/tools/` and reusable external toolkits in `src/codemie_tools/`, the two packages Poetry ships. Evidence: `pyproject.toml::packages`
- R2. Call GitLab, Jira and cloud SDKs from a service or a `codemie_tools` adapter and return normalized tool output, not provider objects. Evidence: `src/codemie_tools/base/codemie_tool.py::CodeMieTool._run`
- R3. Use `tool_registry.lookup(name)` instead of a new scan or name list of tool classes. Evidence: `src/codemie_tools/base/tool_registry.py::lookup`

## Examples

### Building a ToolRecord for a tool name

Source: `src/codemie_tools/base/tool_registry.py::lookup`

```python
def lookup(name: str) -> ToolRecord | None:
    classes = _CLASSES_BY_NAME.get(name)
    if not classes:
        return None
    return ToolRecord(
        name=name,
        classes=tuple(classes),
        modules=tuple(c.__module__ for c in classes),
        script_callable=tuple(bool(getattr(c, "script_callable", False)) for c in classes),
        step_config_fields=tuple(tuple(getattr(c, "step_config_fields", ())) for c in classes),
        tool_access_filter=tuple(getattr(c, "tool_access_filter", None) for c in classes),
        flags=tuple(frozenset(f for f in _TOOL_FLAGS if getattr(c, f, False) is True) for c in classes),
    )
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A tool name is unknown to `tool_registry.lookup` | `register` skips abstract classes, classes without a default `name`, and classes outside the tool module roots | give the class a default `name` and define it under a tool package | `src/codemie_tools/base/tool_registry.py::register` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Check whether a tool may run in scripts | call `is_script_callable` | `src/codemie_tools/base/tool_registry.py::is_script_callable` |
| Read the class flags of a tool | call `lookup(name)` | `src/codemie_tools/base/tool_registry.py::lookup` |
| Load every tool class before a lookup | call `ensure_loaded` | `src/codemie_tools/base/tool_registry.py::ensure_loaded` |
