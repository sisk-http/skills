---
name: sisk-framework-routing
description: Use this skill for Sisk Framework routing, route parameters, route modules, attributes, prefixes, constraints, and request route metadata.
---
# Sisk Framework - Routing

## Routing

The `Router` maps URLs to callbacks of type `RouteAction`. Ways to define routes:

```csharp
// Inline with Map*
mainRouter.MapGet("/hello/<name>", req => {
    string name = req.RouteParameters["name"].GetString();
    return new HttpResponse($"Hello, {name}");
});

// SetRoute
mainRouter.SetRoute(RouteMethod.Get, "/user/<id>", req => {
    Guid id = req.RouteParameters["id"].GetGuid();
    return new HttpResponse(200);
});

// Route.* helper
mainRouter += Route.Get("/image.png", req => {
    return new HttpResponse { Content = new StreamContent(File.OpenRead("image.png")) };
});
```

**Attributes in classes:**
```csharp
[RoutePrefix("/api/users")]
public class UsersController : RouterModule
{
    [RouteGet]               // GET /api/users
    public HttpResponse List(HttpRequest req) { ... }

    [RouteGet("/<id>")]      // GET /api/users/<id>
    public HttpResponse Get(HttpRequest req) { ... }

    [RoutePost]              // POST /api/users
    public HttpResponse Create(HttpRequest req) { ... }

    [RoutePatch("/<id>")]    // PATCH /api/users/<id>
    public HttpResponse Edit(HttpRequest req) { ... }

    [RouteDelete("/<id>")]   // DELETE /api/users/<id>
    public HttpResponse Delete(HttpRequest req) { ... }
}

mainRouter.SetObject(new UsersController());
```

**Routes with Regex:**
```csharp
mainRouter.SetRoute(new RegexRoute(RouteMethod.Get, @"/uploads/(?<filename>.*\.(jpg|png))", req => {
    string filename = req.RouteParameters["filename"].GetString();
    return new HttpResponse($"File: {filename}");
}));
```

**Router configurations:**
```csharp
mainRouter.MatchRoutesIgnoreCase = true;

mainRouter.NotFoundErrorHandler = () =>
    new HttpResponse(404) { Content = new StringContent("Not found") };

mainRouter.MethodNotAllowedErrorHandler = (ctx) =>
    new HttpResponse(405) { Content = new StringContent("Method not allowed") };

mainRouter.CallbackErrorHandler = (ex, ctx) =>
    new HttpResponse(500) { Content = new StringContent("Internal server error") };
```

Keep exception details in server-side logs, not the client-facing 500 response.

**Wildcard routes:**
```csharp
mainRouter.SetRoute(RouteMethod.Any, "/", handler);           // any method
mainRouter.SetRoute(RouteMethod.Post, Route.AnyPath, handler); // any path
```

- `RouteParameters` and `Query` return `StringValueCollection` — each value is `StringValue` with helpers: `.GetString()`, `.GetGuid()`, `.GetInteger()`, etc.
- Trailing slash is ignored by default. To force: flag `HttpServerFlags.ForceTrailingSlash`.
- Sisk detects route collisions automatically when defining.

---
