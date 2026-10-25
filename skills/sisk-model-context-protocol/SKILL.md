---
name: sisk-model-context-protocol
description: Use when building or reviewing a Sisk.ModelContextProtocol server that exposes MCP tools or Agent Skills over Streamable HTTP.
---
# Sisk.ModelContextProtocol

Use `Sisk.ModelContextProtocol` to expose **Tools** and static **Agent Skills** (`McpSkill`, extension `io.modelcontextprotocol/skills`) over MCP Streamable HTTP. Modern requests are stateless; the supported protocol version is exposed as `McpProvider.PROTOCOL_VERSION` — read the constant, never hard-code a literal — and a legacy `initialize` flow remains for older clients. Prompts, Completions, Logging, and resources other than skill files are not implemented.

```bash
dotnet add package Sisk.ModelContextProtocol --prerelease
```

Read the reference that matches the task before changing code:

- [Setup](./references/setup.md) — when wiring the POST route, `McpProvider`/`UseMcp`, required protocol headers and `_meta`, origin and authentication.
- [Tools and schemas](./references/tools-and-schemas.md) — when defining `McpTool`, input/output schemas, and authorization/validation, or publishing `McpSkill` objects.
- [Runtime and results](./references/runtime-and-results.md) — when writing handlers (context, cancellation, results), handling failures, or checking handled methods, error codes, and legacy compatibility.

For additional details, see the [official Sisk Model Context Protocol documentation](https://docs.sisk-framework.org/docs/extensions/mcp).
