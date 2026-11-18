# MCP Integration

**Project**: codemie · **Stack**: Python FastAPI · **Scope**: MCP configuration routers, toolkit and init timeouts; agent tool adapters are in `agents/agent-tools.md`.

## Overview

MCP servers become toolkits through `MCPToolkitFactory`; listing a server's tools waits at most `MCP_SERVER_INIT_TIMEOUT` and logs how to raise it. Config and auth routers are registered in `main.py`.

| Part | What it holds | Start here |
|---|---|---|
| `src/codemie/service/mcp/` | MCP client, toolkit factory, tools | `src/codemie/service/mcp/toolkit.py::MCPToolkitFactory` |
| `src/codemie/configs/config.py` | MCP timeouts | `src/codemie/configs/config.py::MCP_SERVER_INIT_TIMEOUT` |

## MCP

### Rules
- R1. Change MCP config and auth through the MCP routers and `src/codemie/service/mcp/`; tools come from `MCPToolkitFactory`. Evidence: `src/codemie/service/mcp/toolkit.py::MCPToolkitFactory.get_toolkit`
- R2. Raise `MCP_SERVER_INIT_TIMEOUT` (seconds, CodeMie) and `MCP_CONNECT_INIT_TIMEOUT` (milliseconds, MCP-Connect) together when a server fails only on first start; the smaller budget wins. Evidence: `src/codemie/service/mcp/toolkit.py::MCPToolkitFactory._list_tools_with_init_timeout`
- R3. Add MCP server handshake instructions to a prompt only through `render_instructions_section`, never by inserting `toolkit.instructions` directly (the text is untrusted; the renderer neutralizes it, delimits it with an HMAC nonce and caps it); set `MCP_INSTRUCTIONS_NONCE_SECRET` so delimiters stay stable across instances. Evidence: `src/codemie/service/mcp/server_instructions.py::render_instructions_section`
- R4. Carry MCP server instructions only for servers whose tools the agent keeps: take them from `ToolkitService.collect_agent_tools` (or `MCPToolkitService.get_mcp_server_tools_with_instructions`) and filter with `instructions_for_tools` after the tool list is final (otherwise the prompt describes tools the agent cannot call). Evidence: `src/codemie/service/mcp/server_instructions.py::instructions_for_tools`

### Constraints
| Do not | Do instead | Why | Evidence |
|---|---|---|---|
| Close a block of untrusted text in a prompt with a fixed marker defused by an exact-case `str.replace` | Delimit it with a per-conversation nonce, NFKC-normalize the text, then defuse marker spellings with a case-insensitive pattern, as `render_instructions_section` and `_neutralize` do | a server can emit a lowercase or homoglyph copy of a fixed marker that the model reads as the end of the block | `src/codemie/service/mcp/server_instructions.py::render_instructions_section` |
| Bound third-party text merged into a system prompt only per entry | Also cap the whole section, omit blocks past the cap with one notice line and log a warning, as `render_instructions_section` does with `MAX_SECTION_CHARS` | an unbounded number of capped entries still grows every request's prompt without limit | `src/codemie/service/mcp/server_instructions.py::render_instructions_section` |

## Examples

### Bounding server start with a timeout

Source: `src/codemie/service/mcp/toolkit.py::MCPToolkitFactory._list_tools_with_init_timeout`

```python
    async def _list_tools_with_init_timeout(
        self,
        server_config: MCPServerConfig,
        execution_context: MCPExecutionContext | None,
    ) -> MCPListToolsResponse:
```

## Quick reference

| Task | Do | Where |
|---|---|---|
| Raise the server start budget | set `MCP_SERVER_INIT_TIMEOUT` (and the MCP-Connect side) | `src/codemie/configs/config.py::MCP_SERVER_INIT_TIMEOUT` |
| Raise the per-call budget | set `MCP_CLIENT_TIMEOUT` | `src/codemie/configs/config.py::MCP_CLIENT_TIMEOUT` |
| Build a toolkit for a server | call `get_toolkit` on the factory | `src/codemie/service/mcp/toolkit.py::MCPToolkitFactory.get_toolkit` |
