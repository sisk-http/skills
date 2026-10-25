# Responses and compression

For additional details, see the official Sisk documentation on [responses and compression](https://docs.sisk-framework.org/docs/fundamentals/responses).

Return `HttpResponse` directly for simple endpoints. Register value handlers when a reusable application result type needs centralized conversion.

```csharp
router.RegisterValueHandler<ApiResult>(result =>
    new HttpResponse(result.StatusCode)
        .WithContent(JsonContent.Create(result.Value)));
```

Handler registration order matters because assignable types are considered in order. Register specific types before an `object` fallback. `RegisterValueHandler<T>` throws when `T` is already registered or is `HttpResponse`. Route results must not resolve to `null`.

## Final headers and response transforms

Use `context.ExtraHeaders` to add headers without replacing headers the response already set. Use `context.OverrideHeaders` to replace them; it is applied after `ExtraHeaders` and wins.

An `AfterResponse` request handler can inspect `context.RouterResponse` and return a replacement. Preserve status, required headers, cookies, and content semantics deliberately. Prefer public content APIs; do not reach into private framework fields to avoid a copy.

Centralize client-safe error conversion in `Router.CallbackErrorHandler` when `ThrowExceptions` is disabled. Log unexpected details internally and return a stable generic error to clients.

## Compression

Enable automatic response compression explicitly:

```csharp
builder.UseConfiguration(configuration =>
{
    configuration.EnableAutomaticResponseCompression = true;
});
```

Sisk negotiates Brotli, GZip, or Deflate (in that order) from the request when the response is eligible.

To compress one response without enabling automatic compression, inspect `Accept-Encoding` and wrap the content directly:

```csharp
HttpContent content = JsonContent.Create(payload);
var accepted = request.Headers.AcceptEncoding;

if (accepted?.Contains("br", StringComparison.OrdinalIgnoreCase) == true)
    content = new BrotliContent(content);
else if (accepted?.Contains("gzip", StringComparison.OrdinalIgnoreCase) == true)
    content = new GZipContent(content);
else if (accepted?.Contains("deflate", StringComparison.OrdinalIgnoreCase) == true)
    content = new DeflateContent(content);

return new HttpResponse(200).WithContent(content);
```

`BrotliContent`, `GZipContent`, and `DeflateContent` copy the inner content headers and set `Content-Encoding`. Do not send a compressed representation the client did not advertise. Do not pre-compress content and also rely on automatic compression. Streaming responses, SSE, and WebSockets have their own framing and lifecycle; do not treat them as ordinary buffered responses.
