# Request Hedging

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Hedged fast-path tool and provider calls for assistants; the agent runtime is in `agents/langchain-agent-patterns.md`.

## Overview

A hedged assistant races its LLM run against hedge tools: `get_request_handler` returns `HedgedAssistantHandler` when the assistant has `hedging_config` and the `requestHedging` customer feature is on. Hedge tools subclass `CodeMieHedgeTool` and return a `HedgeToolResult`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/rest_api/handlers/hedged_handler.py` | hedged request handler | `src/codemie/rest_api/handlers/hedged_handler.py::HedgedAssistantHandler` |
| `src/codemie_tools/base/codemie_hedge_tool.py` | hedge tool base | `src/codemie_tools/base/codemie_hedge_tool.py::CodeMieHedgeTool` |
| `src/codemie_tools/hedging/` | example hedge toolkit | `src/codemie_tools/hedging/example_hedge_tool.py::ExampleHedgeTool` |

## Request hedging

### Rules
- R1. Use hedging to race a fast-path tool against the agent run; the fast-path answer wins when it arrives within `timeout_ms` (default 200). Evidence: `src/codemie/rest_api/models/hedging.py::HedgingConfig`
- R2. Expect `HedgedAssistantHandler` only when the assistant has `hedging_config` and the `requestHedging` customer feature is enabled; otherwise the standard handler runs. Evidence: `src/codemie/rest_api/handlers/assistant_handlers.py::get_request_handler`
- R3. Write an internal fast path as a `CodeMieHedgeTool` whose `execute` returns `HedgeToolResult(empty=...)`. Evidence: `src/codemie_tools/base/codemie_hedge_tool.py::CodeMieHedgeTool`
- R4. Accept a provider fast-path result through `result_condition` (a boolean expression over the result; unset accepts any non-null result). Evidence: `src/codemie/rest_api/models/hedging.py::HedgingProviderToolDetails`
- R5. Map tool inputs with `input_mapping` Jinja2 templates; forward `{{user.token}}` to trusted internal providers only. Evidence: `src/codemie/rest_api/models/hedging.py::HedgingConfig`
- R6. List hedgeable tools through `get_hedgeable_tools_info`. Evidence: `src/codemie/service/tools/tools_info_service.py::get_hedgeable_tools_info`
- R7. Expect no hedging when the built agent tree requires tool confirmation (`tree_requires_confirmation`): the hedged handler drops its build and runs the standard stream, so pause and resume work as for a standard assistant. Evidence: `src/codemie/rest_api/handlers/hedged_handler.py::HedgedAssistantHandler._handle_stream`

## Examples

### A hedge tool returning a HedgeToolResult

Source: `src/codemie_tools/hedging/example_hedge_tool.py::ExampleHedgeTool`

```python
class ExampleHedgeTool(CodeMieHedgeTool):
    name: str = "example_hedge_tool"
    description: str = "Example hedgeable tool for testing. Randomly returns an empty or non-empty result."

    def execute(self, query: str, metadata: dict[str, Any] | None = None) -> HedgeToolResult:
        if random.random() < 0.5:
            return HedgeToolResult(empty=True)
        return HedgeToolResult(empty=False, data=f"Fast response for query: {query!r}")
```

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a hedge tool | subclass `CodeMieHedgeTool` and implement `execute` | `src/codemie_tools/base/codemie_hedge_tool.py::CodeMieHedgeTool` |
| Turn hedging on for an assistant | set `hedging_config` and enable the `requestHedging` feature | `src/codemie/rest_api/handlers/assistant_handlers.py::get_request_handler` |
| Keep log context in a hedge task | bind it as the handler does | `src/codemie/rest_api/handlers/hedged_handler.py::_bind_request_logging_context` |
