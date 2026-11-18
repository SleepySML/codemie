# Performance Patterns

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: Async I/O and background or streaming work; pagination and batching are in `data/database-optimization.md`.

## Overview

Async code paths take `AsyncSession`s from the async engine; long work runs as workflow executions or background tasks that report progress through a `ThoughtQueue`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/clients/postgres.py` | sync and async engines and sessions | `src/codemie/clients/postgres.py::get_async_session` |
| `src/codemie/core/thought_queue.py` | progress queue for background runs | `src/codemie/core/thought_queue.py::ThoughtQueue` |
| `src/codemie/rest_api/handlers/hedged_handler.py` | hedged parallel calls | `src/codemie/rest_api/handlers/hedged_handler.py::copy_logging_context` |

## Performance

### Rules
- R1. Use `get_async_session` on async paths and run blocking work off the event loop with `asyncio.to_thread`, as the A2A task manager does. Evidence: `src/codemie/rest_api/a2a/server/codemie_task_manager.py::to_thread`
- R2. Run long work as a workflow execution or background task that reports through a `ThoughtQueue`; `create_executor` creates one when no queue is passed. Evidence: `src/codemie/workflows/workflow.py::WorkflowExecutor.create_executor`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Query from an async route | use `get_async_session` | `src/codemie/clients/postgres.py::get_async_session` |
| Give ledger reads their own pool | set `SPEND_LEDGER_POOL_SIZE`; reads use `get_ledger_async_session` | `src/codemie/clients/postgres.py::get_ledger_async_session` |
| Run a workflow in background | create the executor without a queue; it creates a `ThoughtQueue` | `src/codemie/workflows/workflow.py::WorkflowExecutor.create_executor` |
