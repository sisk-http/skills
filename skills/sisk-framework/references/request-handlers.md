# Request handlers

For additional details, see the official Sisk documentation on [request handlers](https://docs.sisk-framework.org/docs/fundamentals/request-handlers).

Request handlers are route middleware. Use `RequestHandler` for synchronous checks and `AsyncRequestHandler` for I/O.

```csharp
public sealed class RequireJson : RequestHandler
{
    public override HttpResponse? Execute(HttpRequest request, HttpContext context) =>
        request.Headers.ContentType?.Contains("application/json") == true
            ? Next()
            : new HttpResponse(415);
}
```

`Next()` is explicit `null`: continue the pipeline. A non-null response stops the current pipeline and becomes the result.

## Two-phase order

For a matched route, Sisk executes:

1. global `BeforeResponse` handlers in array order;
2. route/module `BeforeResponse` handlers in array order;
3. the route action;
4. global `AfterResponse` handlers in array order;
5. route/module `AfterResponse` handlers in array order.

Handlers whose `ExecutionMode` does not match the current phase are skipped. Keep arrays ordered even when before- and after-response handlers are interleaved.

A non-null before-response result skips the route action and later groups. A non-null after-response result replaces the outgoing response and stops the remaining handlers. An after-response handler can inspect the original route result through `context.RouterResponse`.

```csharp
public sealed class AddResponseHeader : RequestHandler
{
    public override RequestHandlerExecutionMode ExecutionMode { get; init; } =
        RequestHandlerExecutionMode.AfterResponse;

    public override HttpResponse? Execute(HttpRequest request, HttpContext context)
    {
        context.ExtraHeaders.Add("X-Request-Policy", "standard");
        return Next();
    }
}
```

A single handler can run in both phases. `context.RouterResponse` is `null` before the route action and contains the route result afterward:

```csharp
public sealed class MeasureRequest : RequestHandler
{
    public override RequestHandlerExecutionMode ExecutionMode { get; init; } =
        RequestHandlerExecutionMode.Both;

    public override HttpResponse? Execute(HttpRequest request, HttpContext context)
    {
        if (context.RouterResponse is null)
        {
            context.RequestBag.Set(Stopwatch.StartNew());
            return Next();
        }

        Stopwatch stopwatch = context.RequestBag.Get<Stopwatch>();
        stopwatch.Stop();
        context.ExtraHeaders.Add("Server-Timing", $"app;dur={stopwatch.Elapsed.TotalMilliseconds:F1}");
        return Next();
    }
}
```

Prefer `ExtraHeaders` for additive final headers. When replacing a response, preserve status and headers intentionally; do not depend on private `HttpContent` fields or runtime-specific reflection tricks.

## Registration scopes

- `Router.GlobalRequestHandlers`: every route in the router.
- `Route.RequestHandlers`: one route.
- `RouterModule.HasRequestHandler`: every route created from the module.
- `[RequestHandler<T>]`: class- or method-level activation, with constructor arguments when needed.
- `RequestHandler.Create` and `AsyncRequestHandler.Create`: small inline policies.

`Route.BypassGlobalRequestHandlers` compares handler instances by reference. Reuse the same global instance when a route must bypass it.

Pass authenticated principals and request-scoped state to later handlers through typed `RequestBag` entries; see [Application architecture](./application-architecture.md). For counters shared by concurrent work in one request, `context.Interlocked` (1.7) offers atomic `Add`, `Exchange`, `CompareExchange`, and `Inspect` over named `double` values stored in the bag.

When `ThrowExceptions` is disabled, handler exceptions can flow through `Router.CallbackErrorHandler`. Throw only failures the centralized policy understands; do not use exceptions as an undocumented replacement for normal handler responses.
