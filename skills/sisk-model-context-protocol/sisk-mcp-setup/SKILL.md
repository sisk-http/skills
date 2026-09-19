---
name: sisk-mcp-setup
description: Use this skill for Sisk.ModelContextProtocol quick setup, standalone provider wiring, and UseMcp singleton configuration.
---
# Sisk.ModelContextProtocol - Setup

## Quick Setup

Choose a standalone provider for multiple independent endpoints; use `UseMcp` for one app-wide provider. The endpoint path is arbitrary. This implementation handles GET initialization and POST calls; `MapAny` below routes both.

Protect the route with the application's authentication and authorization before exposing private or mutating tools. MCP input schemas do not provide access control.

### Standalone provider (multiple providers or manual wiring)

```csharp
using Sisk.Core.Http;
using Sisk.ModelContextProtocol;
using LightJson.Schema;

var mcp = new McpProvider(
    serverName:    "my-server",
    serverTitle:   "My MCP Server",
    serverVersion: new Version(1, 0));

mcp.Tools.Add(new McpTool(
    name:             "say_hello",
    description:      "Says hello to someone.",
    schema:           JsonSchema.CreateObjectSchema(
        properties: new Dictionary<string, JsonSchema>
        {
            { "name", JsonSchema.CreateStringSchema(description: "Person's name.") }
        },
        requiredProperties: ["name"]),
    executionHandler: async ctx =>
    {
        string name = ctx.Arguments["name"].GetString();
        return McpToolResult.CreateText($"Hello, {name}!");
    }));

using var host = HttpServer.CreateBuilder()
    .UseListeningPort(5000)
    .UseRouter(r =>
    {
        r.MapAny("/mcp", async req => await mcp.HandleRequestAsync(req));
    })
    .Build();

await host.StartAsync();
```

### Singleton via builder (`UseMcp`)

```csharp
using var host = HttpServer.CreateBuilder()
    .UseListeningPort(5000)
    .UseMcp(mcp =>
    {
        mcp.ServerName  = "my-server";
        mcp.ServerTitle = "My MCP Server";

        mcp.Tools.Add(new McpTool(
            name:             "say_hello",
            description:      "Says hello.",
            schema:           JsonSchema.Empty,
            executionHandler: ctx => Task.FromResult(McpToolResult.CreateText("Hello!"))));
    })
    .UseRouter(r =>
    {
        // HandleMcpRequestAsync uses the singleton configured by UseMcp
        r.MapAny("/mcp", async req => await req.HandleMcpRequestAsync());
    })
    .Build();
```

- `UseMcp(McpProvider)` — registers an existing provider instance as the singleton.
- `UseMcp(Action<McpProvider>)` — creates and configures a new provider as the singleton.
- `request.HandleMcpRequestAsync()` — dispatches to the singleton; throws `InvalidOperationException` if `UseMcp` was never called.

---
