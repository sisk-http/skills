# Routing

For additional details, see the official Sisk documentation on [routing](https://docs.sisk-framework.org/docs/fundamentals/routing), [matching precedence](https://docs.sisk-framework.org/docs/fundamentals/routing#matching-precedence), and [custom route matching](https://docs.sisk-framework.org/docs/fundamentals/routing#custom-route-matching).

Use `Map*` for direct routes and attribute modules for larger route groups. Router matching is case-sensitive by default (`MatchRoutesIgnoreCase` is `false`) and `CheckForRouteCollisions` is `true`. Configure both before the server starts:

```csharp
router.MatchRoutesIgnoreCase = true;
```

Set `MatchRoutesIgnoreCase` only when `/Orders` and `/orders` should match the same route. At startup, `CheckForRouteCollisions` rejects static routes with overlapping method and path and, since 1.7, `Route.Name` values used by more than one route (`ArgumentException`). Setting it to `false` disables both checks.

```csharp
router.MapGet("/users/<id>", request =>
{
    Guid id = request.RouteParameters["id"].GetGuid();
    return new HttpResponse($"User {id}");
});
```

`RouteParameters` and `Query` are `StringValueCollection`; use helpers such as `GetString()`, `GetGuid()`, and `GetInteger()`.

## Route types and precedence

Routes created by `Map*`, `Route` instances, and route attributes are static: they follow a path pattern. `RegexRoute`, `PrefixRoute`, and any `Route` subclass that overrides `IsStatic` to `false` are dynamic. Static routes are always tested first, regardless of registration order; dynamic routes keep their registration order. The collision check compares only static routes.

```csharp
router.Map(new RegexRoute(RouteMethod.Get, @"^/items/(?<id>\d+)$", request =>
    new HttpResponse(request.RouteParameters["id"].GetString())));

router.MapQuery("/search", request => new HttpResponse(request.Body));
router.MapFileSystem("/assets", "wwwroot");
```

- Regex patterns are not anchored automatically; use `^` and `$` to match the whole path. `Route.UseRegex` is read-only and obsolete: create a `RegexRoute` instead. The `[RegexRoute]` attribute and `RouteAttribute.UseRegex` still work.
- `PrefixRoute` matches its path and everything below it by whole segments. `MapFileSystem` (not `MapFileServer`) creates one and is not affected by `ForceTrailingSlash`.
- `RouteMethod.Any` includes `Query` (HTTP `QUERY`). Use `MapQuery`, `Route.Query`, or `[RouteQuery]`; `HttpHeaderCollection.AcceptQuery` advertises accepted media types.
- Custom matching: subclass `Route` and override `Match(string requestPath, Router router)`, returning `RouteMatch.NotMatched` or `new RouteMatch(true, parameters)`. Override `IsStatic` to `false`, and `AllowRewrites` to `false` when `Path` is not a path pattern. `Match` runs concurrently for every request, tests only the path (the method is checked afterwards), and should honor `router.MatchRoutesIgnoreCase`.

## Attribute modules

```csharp
[RoutePrefix("/api/users")]
public sealed class UsersModule : RouterModule
{
    public UsersModule() => HasRequestHandler(new AuditRequest());

    [RouteGet("/<id>")]
    [RouteMetadata("OPERATION", "READ_USER")]
    public HttpResponse Get(HttpRequest request) => new HttpResponse(200);
}
```

Call `HasPrefix` and `HasRequestHandler` from the constructor. `OnSetup(Router)` runs lazily while the module's first route is being created, after that route's handler list and the module prefix were read, so handlers or a prefix set there miss the first route. Use `OnSetup` for router-level work, and always call `base.OnSetup(parentRouter)` in an override. `OnRouteCreating` can adjust each generated `Route` before registration.

Register modules explicitly or by scanning:

```csharp
router.MapInstance<UsersModule>();
router.AutoScanModules<ApiModule>();
```

`MapInstance<T>()` (requires `new()`) and `SetObject<T>(instance)` carry trimming annotations; `MapInstance(object)` is annotated `RequiresUnreferencedCode`. `AutoScanModules<TBase>()` scans `TBase`'s assembly, instantiates non-abstract subclasses with `Activator.CreateInstance` (parameterless constructor), and maps only static methods of abstract ones. It is not compatible with Native AOT/trimming.

Handlers apply in this order: class-level `[RequestHandler]`, module `HasRequestHandler`, method-level `[RequestHandler]`. Class-level metadata is applied to every route; method-level metadata extends the specific route.

## Failures

A route action must return a non-null reference type that resolves to an `HttpResponse`; see value handlers in [Responses and compression](./responses-and-compression.md). `ThrowExceptions` defaults to `false`; in that mode, exceptions from routes and request handlers go to `Router.CallbackErrorHandler`, and without one the server answers `500`. `NotFoundErrorHandler` and `MethodNotAllowedErrorHandler` have defaults; replace them for consistent 404 and 405 bodies. Return client-safe messages and write exception details to the error log.
