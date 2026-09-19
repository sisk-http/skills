---
name: sisk-mcp-runtime
description: Use this skill for Sisk.ModelContextProtocol runtime context, tool results, handled MCP methods, error handling, and implementation notes.
---
# Sisk.ModelContextProtocol - Runtime and Results

## McpToolContext

Provided to the execution handler:

```csharp
executionHandler: async (McpToolContext ctx) =>
{
    // required argument
    string query = ctx.Arguments["query"].GetString();

    // optional argument (may be absent/null)
    int limit = ctx.Arguments["limit"].MaybeNull()?.GetInteger() ?? 10;

    // raw JSON object access
    JsonObject raw = ctx.Arguments["options"].GetJsonObject();

    // array
    double[] numbers = ctx.Arguments["numbers"].GetJsonArray().ToArray<double>();

    // metadata from the LLM client (rarely needed)
    JsonObject meta = ctx.Metadata;

    // cancel long-running work on client disconnect
    ctx.Cancellation.ThrowIfCancellationRequested();

    return McpToolResult.CreateText("done");
}
```

| Property | Type | Description |
|---|---|---|
| `Arguments` | `JsonObject` | Validated input arguments from the tool call. |
| `Metadata` | `JsonObject` | Optional `_meta` field from the protocol request. |
| `ToolName` | `string` | Name of the invoked tool. |
| `Server` | `McpProvider` | The owning `McpProvider` instance. |
| `Request` | `HttpRequest` | The raw Sisk `HttpRequest` (access headers, IP, etc.). |
| `Cancellation` | `CancellationToken` | Raised on client disconnect or timeout. |

**Reading argument values (LightJson API):**

| Method | Returns |
|---|---|
| `.GetString()` | `string` — throws if null |
| `.GetNumber()` | `double` |
| `.GetInteger()` | `int` |
| `.GetBoolean()` | `bool` |
| `.GetJsonObject()` | `JsonObject` |
| `.GetJsonArray()` | `JsonArray` |
| `.MaybeNull()` | `JsonValue?` — returns `null` if value is JSON null |

---

## McpToolResult

```csharp
// text (most common)
McpToolResult.CreateText("The answer is 42.");

// image — bytes are base64-encoded automatically
byte[] png = await screenshot.CaptureAsync();
McpToolResult.CreateImage(png, "image/png");

// audio
byte[] wav = await audio.RecordAsync();
McpToolResult.CreateAudio(wav, "audio/wav");

// combine multiple content pieces into one response
McpToolResult.Combine(
    McpToolResult.CreateText("Here is the screenshot:"),
    McpToolResult.CreateImage(png, "image/png"),
    McpToolResult.CreateText("Analysis complete.")
);
```

- `Combine` with a single argument returns it directly (no wrapping).
- `Combine` with multiple arguments merges them into a JSON array.

---

## Handled MCP Methods

The following JSON-RPC methods are handled by `HandleRequestAsync`:

| Method | Notes |
|---|---|
| `GET` (HTTP) | Returns an `initialize` response (capabilities). |
| `initialize` | Handshake — returns server info and capabilities. Protocol version mismatch is tolerated. |
| `tools/list` | Returns the registered `Tools` list. |
| `tools/call` | Validates schema, invokes `ExecuteAsync`, returns result. |
| `ping` | Returns empty `{}`. |
| `notifications/*` | Accepted silently (`202 Accepted`). |
| anything else | Returns JSON-RPC error `-32000 "Method not found or not supported"`. |

---

## Error Handling

Exceptions thrown inside a tool handler are **caught automatically** and returned as text with `isError: true`, including the exception message. Avoid exposing credentials or internal details in those messages.

`McpToolResult.CreateText("Operation timed out...")` alone is an ordinary success result, not an error. When customizing failures, preserve `isError: true` using the installed version's supported API; do not guess an overload or turn cancellation into success.

---

## Notes

- See [setup](../sisk-mcp-setup/SKILL.md) for endpoint wiring and singleton versus multiple providers.
- Tools are registered once at startup; the tool list does not hot-reload (`listChanged: false`).
- Schema validation checks argument shape, not authorization or business constraints. Enforce those before side effects, and pass `ctx.Cancellation` to long-running operations.
- For AOT/trimming support, be aware that `JsonSchema` validation uses reflection internally.
