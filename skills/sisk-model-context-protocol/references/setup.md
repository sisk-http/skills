# Setup

For additional details, see the [official Sisk Model Context Protocol documentation](https://docs.sisk-framework.org/docs/extensions/mcp).

## Current transport

`McpProvider` implements singular request/response [Streamable HTTP](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http). Read the current version from `McpProvider.PROTOCOL_VERSION`; never copy its literal value into application logic.

Expose the endpoint with **POST only**. `HandleRequestAsync` returns `405 Method Not Allowed` for `GET` and every other method, so use `MapPost` even if an example maps `Any` or `Get`.

```csharp
using LightJson.Schema;
using Sisk.Core.Http;
using Sisk.ModelContextProtocol;

var provider = new McpProvider("catalog", "Catalog tools", new Version(1, 0))
{
    ServerDescription = "Look up catalog entries.",
    OriginValidator = (request, origin) =>
        origin is "https://app.example.com" && HasValidAccessToken(request)
};

provider.Tools.Add(new McpTool(
    name: "find_product",
    description: "Finds a product visible to the authenticated caller.",
    schema: JsonSchema.CreateObjectSchema(
        properties: new Dictionary<string, JsonSchema>
        {
            ["sku"] = JsonSchema.CreateStringSchema(description: "Product SKU.")
        },
        requiredProperties: ["sku"]),
    executionHandler: async ctx =>
    {
        var caller = await RequireCallerAsync(ctx.Request, ctx.Cancellation);
        string sku = ctx.Arguments["sku"].GetString();
        return McpToolResult.CreateText(await FindProductAsync(caller, sku, ctx.Cancellation));
    }));

using var host = HttpServer.CreateBuilder()
    .UseListeningPort(5000)
    .UseRouter(router => router.MapPost("/mcp", async request =>
        await provider.HandleRequestAsync(request, request.DisconnectToken)))
    .Build();

await host.StartAsync();
```

`UseMcp(provider)` (or `UseMcp(Action<McpProvider>)`, which creates a new provider) sets the process-wide provider used by `request.HandleMcpRequestAsync(cancellation)`. A later call replaces that singleton; it does not register a provider per route. The request extension throws `InvalidOperationException` when no provider was configured. Call `provider.HandleRequestAsync` directly for multiple providers or explicit route ownership.

`ServerName`, `ServerTitle`, `ServerVersion`, `ServerDescription`, `ServerWebsiteUrl`, `ClientInstructions` (sent as `instructions`), and `Icons` are advertised through discovery and initialization metadata; treat them as public client-facing values. Defaults are placeholders (`ExampleServer`, `Example Server Display Name`, and the description `An example MCP server providing tools`, which the three-argument constructor does not override), so always set them.

## Modern client contract

A request is treated as modern when it has `params._meta.io.modelcontextprotocol/protocolVersion`, calls `server/discover`, or sends an `MCP-Protocol-Version` header that is not a legacy version. Modern requests bind the protocol version to `McpProvider.PROTOCOL_VERSION`. The provider validates:

- `MCP-Protocol-Version`;
- `Mcp-Method` against the JSON-RPC method;
- `Mcp-Name` against `params.name` for `tools/call`;
- `params._meta.io.modelcontextprotocol/protocolVersion`; and
- `params._meta.io.modelcontextprotocol/clientCapabilities` as an object.

`Mcp-Name` supports the `=?base64?...?=` encoded form. Missing metadata returns `-32602`, header/metadata mismatches `-32020`, and an unsupported version `-32022`, all with HTTP `400`. Modern responses include `_meta.io.modelcontextprotocol/serverInfo` and do not issue `Mcp-Session-Id`. The provider does not validate `Accept`.

## Origin and authentication

Set `OriginValidator` whenever browser-originated use is possible. Requests without `Origin` are accepted; a request that includes `Origin` receives HTTP `403` unless the validator accepts it, so a `null` validator rejects every browser origin. Origin validation is a browser/CSRF boundary, not identity.

Authenticate every request and authorize each operation and tenant in an upstream Sisk handler or the tool handler. Keep credentials out of tool arguments and results, derive caller and tenant server-side, and return only data the caller may access.
