# Configuration Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Central config, feature flags and dynamic settings; secret handling is in `development/security-patterns.md`.

## Overview

Settings are fields of the pydantic `Config` in `src/codemie/configs/config.py`, read from the process environment, `.env` and `.env.local`; `load_env_files` also fills `os.environ` for modules that read variables directly. Runtime-togglable values go through `DynamicConfigService`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/configs/config.py` | settings object and env file loading | `src/codemie/configs/config.py::Config` |
| `src/codemie/service/dynamic_config_service.py` | admin-editable runtime values | `src/codemie/service/dynamic_config_service.py::DynamicConfigService` |
| `config/` | YAML trees: LLM providers, customer config, katas | `src/codemie/configs/config.py::MODELS_ENV` |

## Configuration

### Rules
- R1. Add a setting as a typed field of `Config` and read it from `config`; values come from the environment, then `.env.local`, then `.env`. Evidence: `src/codemie/configs/config.py::load_env_files`
- R2. Check a feature flag where the app, router or service is assembled, and reach enterprise features through the enterprise loaders, not by assuming the package. Evidence: `src/codemie/rest_api/main.py::ENABLE_USER_MANAGEMENT`
- R3. Read a runtime-togglable flag such as chat contextual naming through `DynamicConfigService` with a safe default; on a naming failure or with the flag off the conversation keeps its truncated first-message name. Evidence: `src/codemie/service/chat_naming_service.py::_is_chat_contextual_naming_enabled`
- R4. Gate a feature whose backend is optional in standalone with an availability function of `cfg` that returns the existing answer unchanged when `STANDALONE` is false and checks only settings (no network call), as `retrieval_available` does; when the UI must hide the feature, publish the result as a runtime component in `CustomerConfig._get_runtime_config`. Evidence: `src/codemie/configs/config.py::mcp_connect_available`
- R5. Gate MCP tool loading on `mcp_connect_available(config)`, not `config.MCP_CONNECT_ENABLED`: in standalone it also needs `MCP_CONNECT_URL`, and without it `ToolkitService._get_tools` logs a warning and loads no MCP tools for an assistant with MCP servers. Evidence: `src/codemie/service/tools/toolkit_service.py::ToolkitService._get_tools`
- R6. Gate a new `customer_config` feature that several routers and a service expose through one shared gate module: an `ensure_<feature>_enabled()` raising 403 for service chokepoints and a router dependency list with `authenticate` first, so an unauthenticated caller gets 401, not 403. Older features (`subWorkflow`, `cliAnalytics`, `costCenters`) still check `customer_config.is_feature_enabled` inline per call site. Evidence: `src/codemie/service/budget/feature_gate.py::ensure_budget_management_enabled`

### Constraints
| Do not | Do instead | Why | Evidence |
|---|---|---|---|
| Do not call a feature gate after an early return, or only in one router when another path serves the same feature | Do instead call it first in the service chokepoint, before any return, as `get_cost_center_budget_summary` calls `ensure_budget_management_enabled` | the empty-input path and the other entry points served the feature with the flag off | `src/codemie/service/budget/cost_center_budget_summary.py::get_cost_center_budget_summary` |

## Examples

### Env file precedence

Source: `src/codemie/configs/config.py::load_env_files`

```python
def load_env_files(root: Path) -> None:
    """Populate os.environ from .env.local and .env without overriding what is already set.

    Precedence: process environment > .env.local > .env. With override=False the first value set wins,
    so .env.local is loaded before .env.
    """
    load_dotenv(root / ".env.local", override=False)
    load_dotenv(root / ".env", override=False)
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A value in `.env` is ignored | the process environment wins over `.env.local`, which wins over `.env` | unset the exported variable or move the value to `.env.local` | `src/codemie/configs/config.py::load_env_files` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a setting | add a typed field with a default to `Config` | `src/codemie/configs/config.py::Config` |
| Read a runtime flag safely | call `DynamicConfigService.get_bool_value_safe` with a default | `src/codemie/service/dynamic_config_service.py::DynamicConfigService.get_bool_value_safe` |
| Override a value on one machine | put it in `.env.local` | `src/codemie/configs/config.py::load_env_files` |
