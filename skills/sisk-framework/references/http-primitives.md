# HTTP primitives

For additional details, see the official Sisk documentation on [HTTP requests](https://docs.sisk-framework.org/docs/fundamentals/requests) and [HTTP responses](https://docs.sisk-framework.org/docs/fundamentals/responses).

Build responses with `new HttpResponse(status)` plus `WithContent(...)`, `WithHeader(...)`, or `WithStatus(...)`. JSON responses use `JsonContent.Create(value)`; there is no `HttpResponse.Json` helper.

## Read the body once

Choose one body mode for each request:

- Buffering: `Body`, `RawBody`, `GetBodyContents()`, `GetBodyContentsAsync()`, and the form and multipart helpers read the whole body into memory.
- Streaming: `GetRequestStream()`. The JSON helpers also read from this stream when `Content-Length` is known, and buffer only chunked bodies.

Do not mix them. After a buffering helper has run, `GetRequestStream()` and the JSON helpers throw `InvalidOperationException`; after the stream was read, buffering helpers see only what is left of it. Pass cancellation when using async APIs.

```csharp
router.MapPost("/upload", async request =>
{
    Stream body = request.GetRequestStream();
    await SaveUploadAsync(body, request.DisconnectToken);
    return new HttpResponse(204);
});
```

## Body size limit

`HttpServerConfiguration.MaximumContentLength` defaults to 100 MB since 1.7 (it was `0`, unlimited).

- A declared `Content-Length` above the limit receives `413` before the route runs.
- A chunked or streamed body is checked while being read, through `RawBody`, `Body`, or `GetRequestStream()`. Going over throws `InsufficientMemoryException`, which becomes `500` unless the application handles it.
- The same value caps WebSocket message size.
- `0` removes the limit; buffering helpers still cannot exceed `Array.MaxLength`. `UsePortableConfiguration` with a JSON file that has a `Server` section assigns `0` when `Server.MaximumContentLength` is omitted.

## JSON and AOT

`GetJsonContent<T>()` and `GetJsonContentAsync<T>()` accept serializer options and default to the static `HttpRequest.DefaultJsonSerializerOptions` (`JsonSerializerDefaults.Web`). A source-generated context can be installed there:

```csharp
[JsonSerializable(typeof(OrderInput))]
internal partial class AppJsonContext : JsonSerializerContext
{
}

HttpRequest.DefaultJsonSerializerOptions = new JsonSerializerOptions(JsonSerializerDefaults.Web)
{
    TypeInfoResolver = AppJsonContext.Default
};
```

This setting is process-wide, not scoped to one server. The context must include every DTO consumed by the default JSON helpers; do not mutate the options after requests begin.

For Native AOT or trimming, pass the generated `JsonTypeInfo<T>` directly; the options-based overloads are annotated `RequiresDynamicCode` and `RequiresUnreferencedCode`:

```csharp
OrderInput? order = await request.GetJsonContentAsync(
    AppJsonContext.Default.OrderInput,
    request.DisconnectToken);
```

## Cancellation and response streams

`DisconnectToken` forwards the engine's abort token. The default `HttpListener` engine does not detect disconnects, so the token is `CancellationToken.None` there; Cadente (see the `sisk-cadente` skill) implements it. Even then it is cooperative: pass it to async calls and dispose streams and resources deterministically. For SSE and WebSockets, detect dead peers with a ping policy (see [Realtime](./realtime.md)).

Use `GetResponseStream()` only when the route owns a streaming response. A request can enter only one streaming mode: response stream, SSE, or WebSocket.
