# LangGraph Workflows

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: LangGraph workflow executor, nodes and transitions; service orchestration is in `architecture/service-layer-patterns.md`.

## Overview

`WorkflowExecutor` turns a `WorkflowConfig` into a LangGraph `StateGraph`: it creates a node per state, wires transitions (single target, parallel fan-out, condition, switch) and compiles the graph. Nodes subclass `BaseNode` and implement `execute` and `get_task`; autonomous workflows run on `SupervisorWorkflowExecutor`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/workflows/workflow.py` | executor: graph build, edges, run loop | `src/codemie/workflows/workflow.py::WorkflowExecutor` |
| `src/codemie/workflows/nodes/` | node types | `src/codemie/workflows/nodes/base_node.py::BaseNode` |
| `src/codemie/workflows/supervisor_workflow.py` | autonomous mode executor | `src/codemie/workflows/supervisor_workflow.py::SupervisorWorkflowExecutor` |
| `src/codemie/workflows/validation/` | workflow config validation | `src/codemie/workflows/workflow.py::WorkflowExecutor.validate_workflow` |

## Key abstractions

| Abstraction | What it is | Where |
|---|---|---|
| BaseNode | abstract node: `execute` does the work, `get_task` builds the task text, `__call__` runs and updates state | `src/codemie/workflows/nodes/base_node.py::BaseNode.execute` |

## Workflows

### Rules
- R1. Extend `WorkflowExecutor` (graph build, transitions, validation) instead of a separate graph execution path; autonomous mode runs on `SupervisorWorkflowExecutor`. Evidence: `src/codemie/workflows/workflow.py::WorkflowExecutor.validate_workflow`
- R2. Add node behaviour as a `BaseNode` subclass implementing `execute` and `get_task`, and route transitions through the executor's transition handlers. Evidence: `src/codemie/workflows/nodes/base_node.py::BaseNode.execute`
- R3. Do not put `next.iter_key` in the same next block as `condition` or `switch` (the execution config schema rejects it); instead route the condition to a no-LLM dispatch state, a `transform_node` custom node extracting the list from `context_store`, whose next holds `state_id` plus `iter_key`. Evidence: `src/codemie/workflows/execution_config_schema.yaml::iter_key`, `config/templates/workflow/aice_java_migration_template.yaml::files-dispatcher`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Add a node type | subclass `BaseNode` and register it where nodes are initialized | `src/codemie/workflows/workflow.py::WorkflowExecutor.initialize_node` |
| Change transition handling | edit the transition handlers | `src/codemie/workflows/workflow.py::WorkflowExecutor._process_transition` |
| Validate a workflow config | call `validate_workflow` | `src/codemie/workflows/workflow.py::WorkflowExecutor.validate_workflow` |
