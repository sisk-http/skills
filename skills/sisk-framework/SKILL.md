---
name: sisk-framework
description: Use when writing or debugging application code on Sisk.HttpServer 1.7: routing and modules, request and server handlers, host builder and bootstrappers, request and response APIs, deferred actions, WebSockets, SSE, and logging. Cadente, Monitoring, MCP, JSON-RPC, and Sisk.Documenting have their own skills.
---
# Sisk Framework

Sisk applications compose `HttpServer`, `Router`, `HttpRequest`, and `HttpResponse` explicitly; MVC-style modules are optional. Read only the reference that matches the change. The references describe Sisk.Core 1.7.

- [Application architecture](./references/application-architecture.md): read when structuring modules, a shared module base, request-scoped dependencies (`RequestBag`), or route metadata.
- [Routing](./references/routing.md): read when mapping routes, parameters, modules, auto-scan/AOT, or using `RegexRoute`, `PrefixRoute`, `QUERY`, custom matching, and router error handlers.
- [Request handlers](./references/request-handlers.md): read when writing route middleware: phases, ordering, short-circuiting, shared state.
- [Server handlers](./references/server-handlers.md): read when hooking server-wide lifecycle events or diagnosing handler exceptions.
- [Hosting and bootstrappers](./references/hosting-and-bootstrappers.md): read when composing the host builder or running startup tasks.
- [Deferred actions](./references/deferred-actions.md): read when work must run after the response is sent.
- [HTTP primitives](./references/http-primitives.md): read when reading request bodies, JSON, size limits, disconnects, or streams.
- [Helpers](./references/helpers.md): read when using `SizeHelper`, `PathHelper`, or `MimeHelper`.
- [Responses and compression](./references/responses-and-compression.md): read when converting route results, setting final headers, or compressing responses.
- [Realtime](./references/realtime.md): read when implementing WebSockets or SSE.
- [LogStream](./references/logstreams.md): read when configuring access/error logs or application log files.

## Behavior changed in 1.7

- `HttpServerConfiguration.MaximumContentLength` defaults to 100 MB (it was unlimited); `0` removes the limit. See [HTTP primitives](./references/http-primitives.md).
- Static routes are matched before `RegexRoute`, `PrefixRoute`, and custom `Route` subclasses; duplicate `Route.Name` values throw at startup; `Route.UseRegex` is obsolete (use `RegexRoute`); `RouteMethod.Any` includes `Query`; `MapFileSystem` creates a `PrefixRoute`. See [Routing](./references/routing.md).
- `HttpServerHandler` exceptions are split between startup-aborting and logged-only hooks. See [Server handlers](./references/server-handlers.md).
- `HttpRequest.DisconnectToken` is `CancellationToken.None` on the default `HttpListener` engine. See [HTTP primitives](./references/http-primitives.md).

For additional details, see the [official Sisk Getting Started documentation](https://docs.sisk-framework.org/docs/getting-started).
