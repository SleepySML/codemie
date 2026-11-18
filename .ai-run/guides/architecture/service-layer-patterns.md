# Service Layer Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Service orchestration and feature service layout; layer boundaries are in `architecture/layered-architecture.md`.

## Overview

Services under `src/codemie/service/` hold business decisions and call repositories, SQL models and providers; routers pass them explicit users and ids. Workflow execution is created through a classmethod factory that picks the executor by workflow mode.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/service/` | feature services, one module or package per feature | `src/codemie/service/assistant_service.py::AssistantService` |
| `src/codemie/workflows/workflow.py` | workflow executor factory | `src/codemie/workflows/workflow.py::WorkflowExecutor.create_executor` |
| `src/codemie/service/llm_service/` | LLM catalogue service | `src/codemie/service/llm_service/llm_service.py::LLMService` |

## Services

### Rules
- R1. Pass services explicit users, ids and command data, not the `Request`; workflow runs start from `WorkflowExecutor.create_executor`. Evidence: `src/codemie/workflows/workflow.py::WorkflowExecutor.create_executor`
- R2. Add behaviour to the feature-scoped service (`AssistantService`, `LLMService`) instead of one broad service, and reuse provider registries. Evidence: `src/codemie/service/llm_service/llm_service.py::LLMService`

## Common pitfalls

| Symptom | Cause | Fix | Evidence |
|---|---|---|---|
| An autonomous workflow ignores code added to `WorkflowExecutor` | `create_executor` returns a `SupervisorWorkflowExecutor` for `WorkflowMode.AUTONOMOUS` | put the change where both executors reach it, or in the supervisor executor too | `src/codemie/workflows/workflow.py::WorkflowExecutor.create_executor` |

## Quick reference

| Task | Do | Where |
|---|---|---|
| Start a workflow execution | call the factory, passing the config, input and user | `src/codemie/workflows/workflow.py::WorkflowExecutor.create_executor` |
| Look up LLM models | use the LLM service, not provider config files | `src/codemie/service/llm_service/llm_service.py::LLMService` |
| Run several writes atomically | wrap them in the transactional session | `src/codemie/clients/postgres.py::transactional_session` |
