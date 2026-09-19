---
name: sisk-framework-handlers
description: Use this skill for Sisk Framework request handlers, middleware flow, response compression, and HttpServerHandler usage.
---
# Sisk Framework - Handlers and Compression

## Request Handlers (Middlewares)

Implement `IRequestHandler` (or inherit `RequestHandler`) to create middlewares:

```csharp
public class AuthHandler : IRequestHandler
{
    public RequestHandlerExecutionMode ExecutionMode { get; init; }
        = RequestHandlerExecutionMode.BeforeResponse;

    public HttpResponse? Execute(HttpRequest request, HttpContext context)
    {
        if (request.Headers.Authorization is null)
            return new HttpResponse(HttpStatusCode.Unauthorized);

        context.RequestBag.Add("User", ResolveUser(request));
        return null; // continue cycle
    }
}
```

`ResolveUser` above is application code: it must validate credentials and reject invalid or unauthorized users. Header presence alone is not authentication.

- Returning `null` → continues cycle.
- Returning `HttpResponse` → interrupts and responds immediately.
- `ExecutionMode.AfterResponse` → overwrites router's response if non-null is returned.

**Association:**

```csharp
// per route
mainRouter.SetRoute(RouteMethod.Get, "/", IndexPage, "", new IRequestHandler[] {
    new AuthHandler(),
    new RateLimitHandler(),
    // ↑ IndexPage executes here
    new AuditHandler()  // AfterResponse
});

// global (all routes)
mainRouter.GlobalRequestHandlers = new IRequestHandler[] { new AuthHandler() };

// by attribute
[RouteGet("/")]
[RequestHandler<AuthHandler>]
public HttpResponse Index(HttpRequest req) { ... }

// attribute with constructor arguments
[RequestHandler<RateLimitHandler>(100, 60)]
public HttpResponse Limited(HttpRequest req) { ... }
```

The attribute example assumes an application-defined `RateLimitHandler(int limit, int windowSeconds)`. C# attribute arguments must be compile-time-compatible values; do not call `TimeSpan.FromMinutes` inside an attribute.

**Bypassing global handler:**
```csharp
var auth = new AuthHandler();
mainRouter.GlobalRequestHandlers = new IRequestHandler[] { auth };

mainRouter.SetRoute(new Route(RouteMethod.Get, "/public", ...) {
    BypassGlobalRequestHandlers = new IRequestHandler[] { auth } // same instance
});
```

> Always use the **same instance** for bypass; creating a new instance won't work.

---

## Response Compression

**Manual per route:**
```csharp
router.MapGet("/hello", req =>
    new HttpResponse {
        Content = new GZipContent(new StringContent("hello")),
        // or BrotliContent, DeflateContent
    });

// with stream
router.MapGet("/archive", req =>
    new HttpResponse {
        Content = new GZipContent(File.OpenRead("/path/to/file.zip"))
        // don't use "using" here — server discards after sending
    });
```

**Automatic (recommended):**
```csharp
config.EnableAutomaticResponseCompression = true;
```

Priority order by `Accept-Encoding`: Brotli (`br`) → GZip (`gzip`) → Deflate (`deflate`).  
Responses that already inherit from `CompressedContent` are not compressed again.

---

## Server Handlers (HttpServerHandler)

`HttpServerHandler` provides hooks into the entire server lifecycle — unlike `IRequestHandler`, it can't be limited to specific routes. A single instance is created per type.

```csharp
public class DatabaseHandler : HttpServerHandler
{
    // called when HTTP connection is opened (headers received)
    protected override void OnHttpRequestOpen(HttpRequest request)
    {
        // read content here if needed after closure
        if (request.Headers["Content-Type"]?.Contains("json") == true)
            _ = request.RawBody; // forces read before GC
    }

    // called when HTTP connection is closed (response sent)
    protected override void OnHttpRequestClose(HttpServerExecutionResult result)
    {
        if (result.Request.Bag.IsSet<DbContext>())
            result.Request.Bag.Get<DbContext>().Dispose();
    }
}
```

**Registration:**
```csharp
using var app = HttpServer.CreateBuilder()
    .UseHandler<DatabaseHandler>()
    .Build();

// or after creation
server.RegisterHandler<DatabaseHandler>();
```

**Extension via extension methods:**
```csharp
public static class DbExtensions
{
    public static DbContext GetDb(this HttpRequest request)
        => request.Bag.GetOrAdd<DbContext>(() => new DbContext());
}
```

Available events (override as needed):
- `OnServerStarting` / `OnServerStopping`
- `OnHttpRequestOpen(HttpRequest)` — connection opened, headers available
- `OnHttpRequestClose(HttpServerExecutionResult)` — response sent, cleanup here
- `OnContextBagCreated(HttpContextBag)`

> Use `HttpServerHandler` to manage request-scoped resources (DB connections, sessions, telemetry). Use `IRequestHandler` for conditional logic per route (authentication, rate limit, validation).
