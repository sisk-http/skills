# Server handlers

For additional details, see the official Sisk documentation on [HTTP server handlers](https://docs.sisk-framework.org/docs/advanced/http-server-handlers).

`HttpServerHandler` observes the whole server, router, and request lifecycle. It is not route middleware and cannot be attached to only one route.

```csharp
public sealed class RequestMetricsHandler : HttpServerHandler
{
    public override int Priority => 100;

    protected override void OnHttpRequestClose(HttpServerExecutionResult result)
    {
        RequestCount.Add(1);
    }
}
```

Register handlers through the host builder or the server API:

```csharp
using var host = HttpServer.CreateBuilder()
    .UseHandler<RequestMetricsHandler>()
    .Build();
```

Lower `Priority` values execute first. A handler instance is server-wide, so shared mutable state must be thread-safe.

Available hooks cover server starting/started/stopping/stopped, router setup, context-bag creation, request open/close, and caught exceptions. These callbacks are synchronous; `AsyncHttpServerHandler` offers async overrides, but the server still blocks on them. Keep per-request hooks fast and move network or storage I/O to the route pipeline, a bounded deferred action, or a durable worker.

Exception rules (1.7): a throw in `OnServerStarting`, `OnServerStarted`, or `OnSetupRouter` aborts startup, wrapped in `InvalidOperationException`. A throw in any other hook is written to `ErrorsLogsStream` and the remaining handlers still run; with `ThrowExceptions` enabled it is rethrown wrapped in `InvalidOperationException`. Exceptions in `OnHttpRequestClose`, `OnException`, and log formatting never reach the HTTP engine and are only logged.

Use server handlers for process-wide metrics, diagnostics, global context initialization, router-wide setup, and server start/stop integration.

By default, Sisk disposes `IDisposable` values in `RequestBag` after deferred actions and before the request-close handler runs. Do not add a second disposal path for the same value unless automatic disposal is disabled and ownership is explicit.
