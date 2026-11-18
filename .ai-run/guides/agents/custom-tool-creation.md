# Custom Tool Creation

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Creating a toolkit and tool, and native connector requirements; toolkit layering is in `agents/tool-overview.md`.

## Overview

A toolkit is a `DiscoverableToolkit` whose `get_definition()` returns a `ToolKit` naming its tools; `toolkit_provider` walks the `codemie_tools` modules and collects `DiscoverableToolkit` subclasses. Each tool is a `CodeMieTool` subclass that registers itself in `tool_registry` when defined.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie_tools/base/` | tool base class, toolkit bases, registry, discovery | `src/codemie_tools/base/base_toolkit.py::DiscoverableToolkit` |
| `src/codemie_tools/notification/` | a small toolkit: email and Telegram | `src/codemie_tools/notification/toolkit.py::NotificationToolkit` |
| `src/codemie_tools/ms_exchange/` | definition-only toolkit with write policy | `src/codemie_tools/ms_exchange/toolkit.py::ExchangeToolkit` |
| `src/external/alembic/versions/` | credential-type enum migrations | `src/external/alembic/versions/5189d4d2afaf_add_ms_exchange_credential_type.py::upgrade` |

## Key abstractions

| Abstraction | What it is | Where |
|---|---|---|
| CodeMieTool | base class; `_run` validates config, times `execute`, limits output and wraps errors in `ToolException` | `src/codemie_tools/base/codemie_tool.py::CodeMieTool._run` |
| DiscoverableToolkit | marker base that makes `toolkit_provider` find a toolkit | `src/codemie_tools/base/toolkit_provider.py::is_toolkit_class` |
| tool_registry | static facts per tool name (`script_callable`, step fields) | `src/codemie_tools/base/tool_registry.py::lookup` |

## Toolkits and tools

### Rules
- R1. Keep new toolkits in `codemie_tools` free of imports from `codemie`; inject host-only dependencies as callables, as `ocr_model_factory` on `EnrichBodyTool` does. Known debt, to fix: src/codemie_tools/base, src/codemie_tools/core, src/codemie_tools/data_management, src/codemie_tools/file_analysis, src/codemie_tools/git, src/codemie_tools/open_api, src/codemie_tools/research. Evidence: `src/codemie_tools/ms_exchange/tools.py::EnrichBodyTool`
- R2. Subclass `DiscoverableToolkit` for a toolkit the platform should find; `toolkit_provider` collects those subclasses only and logs, then skips, a module that fails to import. Evidence: `src/codemie_tools/base/toolkit_provider.py::is_toolkit_class`
- R3. Add tests for a toolkit under the mirrored `tests/codemie_tools/` package; a new config class must appear in `get_available_tools_configs_info` with no error (types without a JSON schema go in `SkipJsonSchema`). Evidence: `tests/codemie_tools/base/test_toolkit_provider.py::test_integration_all_tool_configs_produce_info`
- R4. Read static tool facts from `tool_registry.lookup(name)` and extend `ToolRecord` for a new per-class fact; a `CodeMieTool` subclass registers itself when defined. Evidence: `src/codemie_tools/base/codemie_tool.py::CodeMieTool.__pydantic_init_subclass__`
- R5. Add `CredentialTypes` and `ToolSet` members with the toolkit, and an Alembic migration that calls `op.sync_enum_values` with the full member set (the `CredentialTypes` name is the stored label); new credential types and toolkits take effect after a redeploy of backend and UI. Evidence: `src/external/alembic/versions/5189d4d2afaf_add_ms_exchange_credential_type.py::upgrade`
- R6. Map a new credential config to its type in `__CREDENTIAL_CONFIG_TO_TYPE` and list its secret keys in `LIST_OF_SENSITIVE_FIELDS` so they are masked and encrypted. Evidence: `src/codemie/service/settings/settings.py::LIST_OF_SENSITIVE_FIELDS`
- R7. Build a definition-only toolkit's tools from a saved integration, not from caller-supplied `tool_creds`: `get_tool_with_direct_creds` calls `toolkit.get_tools()`, which such a toolkit does not implement. Evidence: `src/codemie/service/tools/tool_execution_service.py::get_tool_with_direct_creds`
- R8. Retry Exchange writes through `write_retry.run_write` alone; `call_with_retry` is for reads, and a write is resubmitted only after a proven no-mutation rejection. Evidence: `src/codemie_tools/ms_exchange/write_retry.py::run_write`
- R9. Override `is_safe` on a generic HTTP tool to return `self._http_method_is_safe(args)` (GET, HEAD, OPTIONS); the base method returns False, so an un-overridden tool is treated as mutating. Evidence: `src/codemie_tools/access_management/keycloak/tools.py::KeycloakTool.is_safe`
- R10. Return True from `is_safe` on a tool that only reads, as the Elasticsearch search tool does. Evidence: `src/codemie_tools/data_management/elastic/tools.py::SearchElasticIndex.is_safe`

## Examples

### A tool with a config, an input schema and a safe-call override

Source: `src/codemie_tools/notification/telegram/tools.py::TelegramTool`

```python
class TelegramTool(CodeMieTool):
    script_callable = True
    config: TelegramConfig
    name: str = TELEGRAM_TOOL.name
    description: str = TELEGRAM_TOOL.description
    args_schema: Type[BaseModel] = TelegramToolInput

    def is_safe(self, args: dict) -> bool:
        return self._http_method_is_safe(args)

    def execute(self, method: str, relative_url: str, params: Optional[str] = "") -> str:
        """
        Execute a Telegram Bot API request.

        Args:
            method: The HTTP method to use (GET, POST, etc.)
            relative_url: The relative URL of the Telegram API endpoint
            params: JSON string of parameters to send

        Returns:
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A tool config disappears from GET /v1/tools/configs | a `CodeMieToolConfig` field with a type that has no JSON schema makes config info generation log an error and skip the config | wrap such a field in `SkipJsonSchema`, as `FileAnalysisConfig.chat_model` does | `src/codemie_tools/file_analysis/models.py::FileAnalysisConfig` |
| A credential-type migration drops existing enum members | `op.sync_enum_values` sets the enum to exactly the listed values, and the MS Teams migration first listed only some members | list the full current member set plus the new one in upgrade and downgrade | `src/external/alembic/versions/8b2c1a4d5e6f_add_ms_teams_credential_type.py::upgrade` |
| A new toolkit does not appear in the toolkit list | `toolkit_provider` logs an `ImportError` and skips the module, and it collects only `DiscoverableToolkit` subclasses | fix the import and subclass `DiscoverableToolkit` | `src/codemie_tools/base/toolkit_provider.py::is_toolkit_class` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Mark a tool read-only for auto-approval | override `is_safe`; HTTP tools call `_http_method_is_safe` | `src/codemie_tools/notification/telegram/tools.py::TelegramTool.is_safe` |
| Allow a tool in workspace scripts | set `script_callable = True` on the class | `src/codemie_tools/base/codemie_tool.py::CodeMieTool` |
| Add a credential type | add the enum members and an enum-sync migration | `src/external/alembic/versions/5189d4d2afaf_add_ms_exchange_credential_type.py::upgrade` |

## Registry

A new `CodeMieTool` subclass registers itself in `codemie_tools/base/tool_registry.py` automatically; new static facts about tool classes extend `ToolRecord` there, not a new scan.
