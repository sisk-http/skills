# Setup

For additional details, see the [official Sisk JSON-RPC documentation](https://docs.sisk-framework.org/docs/extensions/json-rpc).

```bash
dotnet add package Sisk.JsonRpc
```

Version 1.6.3 targets `net8.0` and depends on `Sisk.HttpServer` 1.6.3 or newer, with no upper bound. Core 1.7.0 raised the default `HttpServerConfiguration.MaximumContentLength` from unlimited to 100 MB; the limit applies to POST bodies and WebSocket messages handled by this extension.

Register methods and map only the transports the application needs (`WebMethod` is in `Sisk.JsonRPC.Annotations`):

```csharp
using var host = HttpServer.CreateBuilder(5555)
    .UseJsonRPC((sender, configuration) =>
    {
        configuration.Handler.Methods.AddMethodsFromType(new MathMethods());
        configuration.Router.MapPost("/rpc", configuration.Handler.Transport.HttpPost);
        configuration.Router.MapGet("/rpc/ws", configuration.Handler.Transport.WebSocket);
    })
    .Build();

await host.StartAsync();

public sealed class MathMethods
{
    [WebMethod]
    public SumResult Sum(double left, double right) => new(left + right);
}

public sealed record SumResult(double Total);
```

The `UseJsonRPC` callback runs when the server starts, once per bound router, all sharing one `JsonRpcHandler`.

## Transports

- `HttpPost`: the standard transport; the body is one request object.
- `HttpGet`: reads `method`, `params` (JSON text) and `id` from the query string, all required. `params` is parsed with default JSON options, not `JsonSerializerOptions`. Expose it only for idempotent methods, since URLs are logged and cached.
- `WebSocket`: map it on a GET route; each message is one request, and the loop ends after 30 minutes without a message.

## Registering methods

- `Methods.AddMethodsFromType(target, prefixTypes: false)` scans public instance and static methods with `[WebMethod]`. Overloads accept a `Type` with an optional target (use no target only for static methods).
- `Methods.AddMethod(name, delegate)` registers a delegate without the attribute; `RemoveMethod(name)` unregisters.
- `[WebMethod("name")]` overrides the method name. With `prefixTypes: true` the name becomes `Type.Method`, and `[WebName("alias")]` on the class replaces the type name.
- Names are case-insensitive; registering a duplicate throws.
- Return type must be a reference type, `Task<T>` or `IAsyncEnumerable<T>` with a reference-type `T`. Value types, including `void`, `double`, structs and `Task<int>`, throw `NotSupportedException` at registration; return a DTO, or wrap the value in `Sisk.Core.Routing.ValueResult<T>` as the error message suggests.
- Scanning and invocation use reflection (`AddMethodsFromType` carries `DynamicallyAccessedMembers`); test trimmed or Native AOT publishes before relying on them.

## Parameters

- Positional arrays must contain between the required and total parameter count. Omitted trailing optional parameters receive `null`/`default(T)`, not the declared default; named objects do apply declared defaults.
- Named objects are matched by name with `JsonSanitizedComparer`: case and non-alphanumeric characters are ignored (`foo_bar` matches `FooBar`). Every property in the object counts toward the maximum, and unknown properties are otherwise ignored.
- A method whose only parameter is named `params` (`@params` in C#) receives the whole `params` value converted to the parameter type, such as `JsonArray` or `JsonObject`.

## Serializer

`JsonRpcHandler.JsonSerializerOptions` is the LightJson `JsonOptions` used for requests, parameters, results and responses. Add converters or set `NamingPolicy` there when application types need it. `SerializationFlags = JsonSerializationFlags.Json5` accepts JSON5 input; it is not JSON-RPC 2.0 interoperable.

## Documentation

Documentation is built from the registered methods when requested, and nothing serves it automatically. Describe methods with `[MethodDescription("text", Category = "...")]` and `[ParamDescription("paramName", "text")]` (both go on the method), then map a route yourself:

```csharp
configuration.Router.MapGet("/rpc/docs", request =>
{
    var documentation = configuration.Handler.GetDocumentation(new JsonRpcDocumentationMetadata
    {
        ApplicationName = "Math API",
        ServicePath = "/rpc",
        AllowedMethods = ["POST"]
    });

    return new HttpResponse
    {
        Content = new HtmlContent(documentation.Export(new JsonRpcHtmlExport()), Encoding.UTF8)
    };
});
```

`GetDocumentation()` without metadata omits the metadata section. `documentation.ExportToJson()` returns JSON text; only the HTML export groups methods by `Category`. Custom formats implement `IJsonRpcDocumentationExporter`. `JsonRpcHtmlExport` exposes `ExportSummary`, `ExportMetadata`, `Header` and `Style`.

Documentation lists every registered method and parameter type and is not an authorization boundary. Map authentication, authorization, rate limiting, and body-size handlers on every JSON-RPC and documentation route; method visibility does not protect a method.
