---
name: sisk-mcp-tools-schema
description: Use this skill for Sisk.ModelContextProtocol McpProvider, McpTool, tool definitions, and JSON schema builder usage.
---
# Sisk.ModelContextProtocol - Tools and Schemas

## McpProvider

| Member | Description |
|---|---|
| `ServerName` | Machine-readable server identifier. Default `"ExampleServer"`. |
| `ServerTitle` | Human-readable display name. Default `"Example Server Display Name"`. |
| `ServerVersion` | `Version` object. Default `1.0`. |
| `Tools` | `IList<McpTool>` — add tools here before the server starts. |
| `PROTOCOL_VERSION` | Const `"2025-06-18"` — the supported MCP spec version. |
| `HandleRequestAsync(HttpRequest, CancellationToken)` | Processes a single MCP HTTP request and returns an `HttpResponse`. |

For endpoint registration, see [setup](../sisk-mcp-setup/SKILL.md). The behavior described here is specific to this library version, not a substitute for the MCP specification.

---

## McpTool

```csharp
new McpTool(
    name:             "tool_name",     // unique, snake_case by convention
    description:      "What it does.", // shown to the LLM
    schema:           JsonSchema.CreateObjectSchema(...),
    executionHandler: async ctx => { ... },
    title:            "Tool Display Name" // optional; defaults to name
)
```

| Property | Type | Description |
|---|---|---|
| `Name` | `string` | Unique tool identifier (shown to the LLM). |
| `Title` | `string?` | Optional display name. Falls back to `Name`. |
| `Description` | `string` | Human/LLM-readable description of what the tool does. |
| `Schema` | `JsonSchema` | Input schema. Use `JsonSchema.Empty` for no arguments. |
| `ExecuteAsync` | `McpToolHandler` | The execution delegate. Can be replaced after construction. |

---

## JSON Schema Builder

Uses `LightJson.Schema.JsonSchema`. All factory methods accept an optional `description` parameter:

```csharp
// object with typed properties
JsonSchema.CreateObjectSchema(
    properties: new Dictionary<string, JsonSchema>
    {
        { "query",   JsonSchema.CreateStringSchema(description: "Search query.") },
        { "limit",   JsonSchema.CreateNumberSchema(description: "Max results.") },
        { "enabled", JsonSchema.CreateBooleanSchema() },
        { "tags",    JsonSchema.CreateArraySchema(
                         itemsSchema: JsonSchema.CreateStringSchema(),
                         minItems: 0,
                         description: "Filter tags.") }
    },
    requiredProperties: ["query"]);   // non-required props may be absent in call

// enum / fixed set of string values
JsonSchema.CreateStringSchema(
    enums: ["asc", "desc"],
    description: "Sort direction.");

// no arguments
JsonSchema.Empty
```

Available factory methods:
- `JsonSchema.CreateObjectSchema(properties, requiredProperties, description)`
- `JsonSchema.CreateArraySchema(itemsSchema, minItems, description)`
- `JsonSchema.CreateStringSchema(enums, description)`
- `JsonSchema.CreateNumberSchema(description)`
- `JsonSchema.CreateBooleanSchema(description)`
- `JsonSchema.Empty`

The schema is automatically validated against tool call arguments before the handler is invoked. Validation failures return a descriptive error text to the LLM without calling the handler.

---
