# Application architecture

For additional details, see the official Sisk documentation on [dependency injection and controller patterns](https://docs.sisk-framework.org/docs/features/instancing).

## MVC-style example with modules

Sisk does not require MVC, `RouterModule`, or a shared API base. Direct `Router.Map*` routes are valid; use modules only when controller-like grouping helps the application.

In an optional MVC-style architecture, a `RouterModule` can group attribute routes, a common prefix, request handlers, and route metadata:

```csharp
[RoutePrefix("/api/orders")]
[RequestHandler<AuthenticateRequest>]
public sealed class OrdersModule : ApiModule
{
    [RouteGet("/<id>")]
    public async Task<HttpResponse> GetOrder()
    {
        Guid id = Request.RouteParameters["id"].GetGuid();
        Order? order = await Orders.FindAsync(id, Request.DisconnectToken);
        return order is null ? new HttpResponse(404) : Json(order);
    }
}
```

Keep modules aligned with stable API boundaries. Put authentication, validation, and response policies in handlers or a shared base instead of repeating them in every route.

### Share request access through a small base module

Parameterless route methods can use the ambient context while the request is executing:

```csharp
public abstract class ApiModule : RouterModule
{
    protected HttpContext Context => HttpContext.Current;
    protected HttpRequest Request => Context.Request;

    protected OrderRepository Orders =>
        Context.RequestBag.GetOrAdd(() => new OrderRepository(Database));

    protected AppDbContext Database =>
        Context.RequestBag.GetOrAdd(AppDbContext.Create);

    protected static HttpResponse Json(object value) =>
        new HttpResponse(200).WithContent(JsonContent.Create(value));
}
```

Keep this base focused on cross-cutting request access. Do not turn it into an inventory of unrelated application services.

## Request-scoped dependency resolution

`RequestBag.GetOrAdd` provides lazy typed storage for one request. This pattern is request-scoped dependency resolution through Sisk's ambient `HttpContext`; it is not an `IServiceProvider` or Microsoft.Extensions.DependencyInjection container.

A before-response handler can resolve authentication data once and store typed state:

```csharp
context.RequestBag.Set(account);
```

The module can require it later:

```csharp
protected Account Account => HttpContext.Current.RequestBag.Get<Account>();
```

By default, Sisk disposes bag values that implement `IDisposable` when the request closes. The current cleanup loop does not automatically call `DisposeAsync` on values that implement only `IAsyncDisposable`. Disable `DisposeDisposableContextValues` only when the application assumes disposal ownership explicitly.

Do not retain `HttpContext`, `HttpRequest`, bag values, or a request-scoped database context after the request lifecycle ends.

## Route metadata as policy input

`[RouteMetadata(key, value)]` copies class- or method-level metadata into `Route.Bag`. Handlers and centralized error or documentation policies can inspect it without embedding business decisions in the router.

```csharp
[RouteMetadata("AUDIENCE", "PUBLIC")]
[RouteGet("/status")]
public HttpResponse Status() => new HttpResponse("ok");
```

Use stable constants for metadata keys. Metadata should describe route policy; request-specific values belong in `RequestBag`.

## Centralize response and error policies

For a consistent API:

- register value handlers for reusable result types;
- use `Router.CallbackErrorHandler` when `ThrowExceptions` is disabled;
- define 404 and 405 handlers centrally;
- expose only expected client-safe failures;
- write internal exception detail to error logs, not response bodies.

Response envelopes, DTO policies, validation libraries, repositories, and expected-error types are application decisions rather than Sisk requirements.
