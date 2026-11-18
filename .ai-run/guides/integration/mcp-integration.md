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

## Examples

### Bounding server start with a timeout

Source: `src/codemie/service/mcp/toolkit.py::MCPToolkitFactory._list_tools_with_init_timeout`

```python
    async def _list_tools_with_init_timeout(
        self,
        server_config: MCPServerConfig,
        execution_context: MCPExecutionContext | None,
    ) -> list[MCPToolDefinition]:
```

## Quick reference

| Task | Do | Where |
|---|---|---|
| Raise the server start budget | set `MCP_SERVER_INIT_TIMEOUT` (and the MCP-Connect side) | `src/codemie/configs/config.py::MCP_SERVER_INIT_TIMEOUT` |
| Raise the per-call budget | set `MCP_CLIENT_TIMEOUT` | `src/codemie/configs/config.py::MCP_CLIENT_TIMEOUT` |
| Build a toolkit for a server | call `get_toolkit` on the factory | `src/codemie/service/mcp/toolkit.py::MCPToolkitFactory.get_toolkit` |
