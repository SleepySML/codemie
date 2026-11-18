# LangChain Agent Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: LangChain agent runtime and result normalization; agent-facing tools are in `agents/agent-tools.md`.

## Overview

`AIToolsAgent` builds an assistant run from its tools, the chat request, callbacks and the request UUID; `TaskResult` is the normalized result a workflow step reads, with structured outputs serialized to strings.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/agents/assistant_agent.py` | the tools agent and its result type | `src/codemie/agents/assistant_agent.py::AIToolsAgent` |
| `src/codemie/agents/langgraph_agent.py` | LangGraph-based agent run | `src/codemie/agents/langgraph_agent.py::LangGraphAgent` |
| `src/codemie/agents/callbacks/` | agent callbacks | `src/codemie/agents/agent_callback_factory.py::build_tool_callbacks` |
| `src/codemie/agents/tools/` | agent-facing tool adapters | `src/codemie/agents/tools/schema_compatibility.py::transform_schema_for_gcp_compatibility` |

## Agent runtime

### Rules
- R1. Extend or configure `AIToolsAgent` and its callback factory instead of a parallel agent runtime for one feature. Evidence: `src/codemie/agents/assistant_agent.py::AIToolsAgent`
- R2. Return agent output through `TaskResult.from_agent_response`, which serializes dict and model results to JSON strings and keeps `intermediate_steps`. Evidence: `src/codemie/agents/assistant_agent.py::TaskResult.from_agent_response`

## Examples

### Normalizing an agent response into a TaskResult

Source: `src/codemie/agents/assistant_agent.py::TaskResult.from_agent_response`

```python
    def from_agent_response(cls, response, truncated: bool = False):
        if isinstance(response, GenerationResult):
            result = response.generated
            # Serialize objects from structured outputs as strings to pass to the next step.
            if isinstance(response.generated, dict):
                result = json.dumps(response.generated)
            elif isinstance(response.generated, BaseModel):
                result = response.generated.model_dump_json()
            return cls(result=result or '', success=response.success, truncated=truncated)

        # Determine the result and success
        if 'output' in response:
            result = extract_text_from_llm_output(response['output'])
            success = True
            if 'intermediate_steps' in response:
                return cls(
                    result=result,
                    success=success,
                    intermediate_steps=response['intermediate_steps'],
                    truncated=truncated,
```

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| A step result is empty and marked failed though the agent ran | `from_agent_response` sets `success=False` and an empty result when the response holds neither `output` nor `generated` | return one of those keys from a custom agent path | `src/codemie/agents/assistant_agent.py::TaskResult.from_agent_response` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Turn an agent response into a step result | call `TaskResult.from_agent_response` | `src/codemie/agents/assistant_agent.py::TaskResult.from_agent_response` |
| Return a failed result | use `TaskResult.failed_result` | `src/codemie/agents/assistant_agent.py::TaskResult.failed_result` |
| Pass callbacks to an agent | give them to the `AIToolsAgent` constructor | `src/codemie/agents/assistant_agent.py::AIToolsAgent` |
