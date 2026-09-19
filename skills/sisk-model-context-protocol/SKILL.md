---
name: sisk-model-context-protocol
description: Use when building or diagnosing a Sisk.ModelContextProtocol server and selecting guidance for setup, tool schemas, execution, or protocol behavior.
---
# Sisk.ModelContextProtocol

Implements an MCP (Model Context Protocol) server over **Streamable HTTP** transport, allowing Sisk applications to expose tools that AI/LLM agent clients can call.

```
dotnet add package Sisk.ModelContextProtocol
```

> **Status: Under development.** Only **Tools** are currently supported. Prompts, Resources, Completions, and server-side Logging are not yet implemented.  
> Protocol version: `2025-06-18`

Reference: https://docs.sisk-framework.org/docs/extensions/mcp  
Namespace: `Sisk.ModelContextProtocol`

---

## Select the relevant skill

Read only the matching skill; go directly to it when the topic is known. This is server-library guidance, not instructions for configuring an MCP client or executing its tools. Confirm capabilities and protocol behavior against the installed package before promising support.

- [sisk-mcp-setup](./sisk-mcp-setup/SKILL.md): quick setup using standalone providers or UseMcp
- [sisk-mcp-tools-schema](./sisk-mcp-tools-schema/SKILL.md): McpProvider, McpTool, and JSON schema builder
- [sisk-mcp-runtime](./sisk-mcp-runtime/SKILL.md): McpToolContext, McpToolResult, handled methods, error handling, and notes

For an implementation, build the affected project and check the changed protocol path with harmless local inputs, including invalid arguments when schemas change. Do not invoke paid, destructive, or external tools merely to validate wiring.
