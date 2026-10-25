# Runtime and results

For additional details, see the [official Sisk Model Context Protocol documentation](https://docs.sisk-framework.org/docs/extensions/mcp).

## Handler context and cancellation

Each handler receives `McpToolContext`:

| Property | Meaning |
| --- | --- |
| `Arguments` | Schema-validated `JsonObject` from `params.arguments`. |
| `Metadata` | The request `params._meta` object. |
| `ToolName` | Invoked tool name. |
| `Request` | Original `HttpRequest`; use it for authentication and request-scoped policy. |
| `Server` | Owning `McpProvider`. |
| `Cancellation` | The cancellation token passed to `HandleRequestAsync`. |

Pass `request.DisconnectToken` to `HandleRequestAsync`, then propagate `ctx.Cancellation` through asynchronous I/O and CPU-bound work. A closed Streamable HTTP connection is a cancellation signal; check the token before expensive loops and do not replace it with `CancellationToken.None`.

```csharp
executionHandler: async ctx =>
{
    string query = ctx.Arguments["query"].GetString();
    int page = ctx.Arguments["page"].MaybeNull()?.GetInteger() ?? 1;

    ctx.Cancellation.ThrowIfCancellationRequested();
    var result = await repository.SearchAsync(query, page, ctx.Cancellation);

    return McpToolResult.CreateText(FormatResult(result));
}
```

## Results

```csharp
McpToolResult text = McpToolResult.CreateText("The answer is 42.");
McpToolResult image = McpToolResult.CreateImage(pngBytes, "image/png");
McpToolResult audio = McpToolResult.CreateAudio(wavBytes, "audio/wav");

McpToolResult combined = McpToolResult.Combine(
    McpToolResult.CreateText("Preview:"),
    image);

var structured = McpToolResult.CreateText("2 results")
{
    StructuredContent = new JsonObject
    {
        ["count"] = 2,
        ["items"] = new JsonArray(["A", "B"])
    }
};
```

`CreateImage` (default `image/png`) and `CreateAudio` (default `audio/wav`) base64-encode bytes. `Combine` returns its sole argument unchanged and flattens multiple content results into one array. A `string` converts implicitly to a text result. When an `OutputSchema` exists, `StructuredContent` must be present and conform to it.

## Exceptions and client-facing failures

Do not expose exception messages, stack traces, credentials, tenant details, or database errors in tool results. The provider behavior is deliberate:

- With `HttpServer` configuration `ThrowExceptions = true`, a handler exception is rethrown.
- Otherwise, the provider writes the exception to `ErrorsLogsStream` when configured and returns the generic error text `Error executing tool` with `isError: true`. This includes `OperationCanceledException` thrown by the handler.

Catch an expected failure only when the tool can safely return an actionable, non-sensitive response; otherwise log with secure server telemetry and let the provider policy apply.

## Protocol behavior

Handled methods: `server/discover`, `tools/list`, `tools/call`, and (modern only) `skills/list`, `skills/get`, `resources/list`, `resources/read`; legacy-only `initialize` and `ping`. Request metadata and headers are required for modern calls as described in [Setup](./setup.md). `server/discover` returns `supportedVersions`, `capabilities`, optional `instructions`, `ttlMs: 0`, `cacheScope: "private"`, and `_meta` server information. Modern `tools/list` and `tools/call` results include `resultType: "complete"` and the same `_meta`. Messages without an `id` are notifications and return `202 Accepted` without a body.

The initialization flow (`initialize`, `ping`, and `Mcp-Session-Id`) remains only for versions in the provider's internal legacy-version list. Do not copy those literals or design new integrations around the legacy path. HTTP `GET` is not an initialization path and returns `405 Method Not Allowed`.

The current error/status behavior includes:

| Condition | JSON-RPC code or result | HTTP status |
| --- | --- | --- |
| Invalid JSON | `-32700` | `400` |
| Invalid envelope (`jsonrpc`, `method`, `id`) | `-32600` | `400` |
| `params` or `_meta` not an object | `-32602` | `400` |
| Missing modern `_meta` protocol version or `clientCapabilities` | `-32602` | `400` |
| Header or metadata mismatch | `-32020` | `400` |
| Unsupported protocol version (`data.supported`/`requested`) | `-32022` | `400` |
| Unknown modern method | `-32601` | `404` |
| Unknown legacy method | `-32601` | `200` |
| `tools/call` with bad `name`/`arguments` shape, or unknown tool | `-32602` | `200` |
| Skill/resource: unknown or non-string `uri`, or `cursor` supplied | `-32602` | `200` |
| Skill catalog inconsistent | `-32603` | `200` |
| Arguments fail the input schema, tool exception, or output-schema mismatch | result with `isError: true` | `200` |

`-32021` is defined by the provider but is not currently emitted. Treat the table as version-sensitive implementation behavior and verify it against the installed package when building a client.
