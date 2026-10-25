# Hosting and bootstrappers

For additional details, see the official Sisk documentation on [building and configuring the HTTP server](https://docs.sisk-framework.org/docs/getting-started#building-the-http-server).

```csharp
using var host = HttpServer.CreateBuilder()
    .UseListeningPort(8080)
    .UseConfiguration(configuration =>
    {
        configuration.ThrowExceptions = false;
        configuration.MaximumContentLength = 8 * SizeHelper.UnitMb;
    })
    .UseBootstraper("db", "Migrate database", MigrateDatabaseAsync)
    .UseBootstraper("db", "Seed data", SeedDatabaseAsync)
    .UseBootstraper("Start workers", StartWorkers)
    .UseRouter(router => router.AutoScanModules<ApiModule>())
    .Build();

await host.StartAsync();
```

The public API is spelled `UseBootstraper`. Overloads: `(Action)`, `(string? name, Action)`, `(string groupKey, string? name, Action)`, and the same three with `Func<Task>`.

## Startup stages

Bootstrappers run inside `HttpServer.Start()`, before routers are bound (so before `UseRouter(Action<Router>)` callbacks) and before the engine listens.

- A bootstrapper without a `groupKey` gets its own group. Groups run in parallel; actions sharing a `groupKey` (case-sensitive) run sequentially in declaration order. Put dependent steps under the same key.
- A failure skips the remaining actions of its group only. After all groups finish, startup aborts with an `InvalidOperationException` whose `InnerException` is an `AggregateException` with one entry per failed bootstrapper.
- Use them for migrations, state repair, serializer setup, worker registration, and dependency validation. Prefer idempotent steps and let a failure stop startup instead of serving from a half-initialized state.

Bootstrappers are not request middleware, shutdown hooks, or job queues. A bootstrapper may start an application-owned worker, but the worker needs its own cancellation, error reporting, and shutdown policy.

## Builder behavior

- The builder composes ports, engines (`UseEngine`), SSL (`UseSsl`), locale, portable configuration, CORS, file server routes (`UseFileServer`), server handlers (`UseHandler`), routing, and startup stages.
- `UseConfiguration` and `UseHttpServer` callbacks run immediately when called, in chain order. `UseRouter(Action<Router>)` runs later, during router setup at start.
- `UseMinimalConfiguration()` is for development and tests: it disables access/error logs, `DisposeDisposableContextValues`, `ForceTrailingSlash`, and automatic compression. Call it before other configuration.
- `Start()` and `StartAsync()` park the calling flow forever by default (`preventHault: true`); `await host.StartAsync()` never completes. Pass `preventHault: false` when code must continue after startup.
