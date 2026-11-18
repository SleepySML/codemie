# A2A Integration

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: A2A protocol version decision, wire shapes and server and client behaviour; other external services are in `integration/external-services.md`.

## Overview

The in-house A2A layer in `src/codemie/rest_api/a2a/` has a server task manager, a client with an agent-card resolver, and a version check that compares Major.Minor with `A2A_PROTOCOL_VERSION`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/rest_api/a2a/server/` | server task manager | `src/codemie/rest_api/a2a/server/codemie_task_manager.py::CodemieTaskManager` |
| `src/codemie/rest_api/a2a/client/` | client and card resolver | `src/codemie/rest_api/a2a/client/card_resolver.py::A2ACardResolver` |
| `src/codemie/rest_api/a2a/version.py` | protocol version check | `src/codemie/rest_api/a2a/version.py::is_supported_protocol_version` |

## A2A protocol

### Rules
- R1. Implement A2A v1.0 in the in-house layer `src/codemie/rest_api/a2a/`; do not migrate to `a2a-sdk` (the transitive 0.3.x package is not the shape reference). Evidence: `src/codemie/rest_api/a2a/version.py::A2A_PROTOCOL_VERSION`
- R2. Handle legacy shapes with read-time before-validators on the in-house types rather than an adapter layer or a data migration. Evidence: `src/codemie/rest_api/a2a/types.py::get_jsonrpc_url`
- R3. Emit camelCase fields with no `kind` discriminator, `SendMessage` results as `{task}` or `{message}`, and serve the card at `/.well-known/agent-card.json`. Evidence: `src/codemie/rest_api/a2a/types.py::A2AProtocolVersionError`
- R4. Read `A2A-Version` from the header, else the query parameter, and reject a missing value or a `Major.Minor` other than 1.0; reject `returnImmediately`, since `SendMessage` blocks until a terminal or input-required state. Evidence: `src/codemie/rest_api/routers/a2a.py::A2A_VERSION_HEADER`
- R5. Send `A2A-Version: 1.0` and raise `A2AProtocolVersionError` when a remote answers as a pre-1.0 agent. Evidence: `src/codemie/rest_api/a2a/client/client.py::A2AProtocolVersionError`
- R6. Refuse to register a card without a JSON-RPC interface declaring 1.0 (`fetch_agent_card` checks it with `is_supported_protocol_version`); v0.3 clients are rejected, not translated. Evidence: `src/codemie/rest_api/a2a/client/card_resolver.py::A2ACardResolver.fetch_agent_card`
- R7. Run the server-side agent with `asyncio.to_thread` so it does not block the event loop. Evidence: `src/codemie/rest_api/a2a/server/codemie_task_manager.py::to_thread`

## Quick reference

| Task | Do | Where |
|---|---|---|
| Check a peer's protocol version | call `is_supported_protocol_version` | `src/codemie/rest_api/a2a/version.py::is_supported_protocol_version` |
| Change server task handling | edit `CodemieTaskManager` | `src/codemie/rest_api/a2a/server/codemie_task_manager.py::CodemieTaskManager` |
| Route an A2A assistant | `get_request_handler` returns `A2AAssistantHandler` for A2A assistants | `src/codemie/rest_api/handlers/assistant_handlers.py::get_request_handler` |
