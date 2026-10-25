# Setup and Access

For additional details, see the [official Sisk documentation](https://docs.sisk-framework.org/docs/) and the [official Sisk.Monitoring package page](https://www.nuget.org/packages/Sisk.Monitoring).

## Registration

```csharp
using Sisk.Monitoring;

var requestCounter = new Counter();
var applicationLog = new LogStream("logs/application.log");

using var server = HttpServer.CreateBuilder()
    .UseMonitoring("/monitoring", monitor => {
        monitor.PageTitle = "Application monitoring";
        monitor.CredentialValidator = credential =>
            ValidateMonitoringCredentialAsync(credential.UserName, credential.Password);
        monitor.CaptureCounter(new MonitoringDefinition<Counter>("HTTP requests", requestCounter) {
            DashboardPinned = true
        });
        monitor.CaptureLogStream(new MonitoringDefinition<LogStream>("Application log", applicationLog));
    })
    .Build();
```

`ValidateMonitoringCredentialAsync` is your own `ValueTask<bool>` check. `CredentialValidator` is `Func<NetworkCredential, ValueTask<bool>>?`.

Overloads (default base path `/monitoring`):

- `UseMonitoring(Action<ApplicationMonitor>)` and `UseMonitoring(string basePath, Action<ApplicationMonitor>)` create a plain `ApplicationMonitor` and configure it.
- `UseMonitoring<T>(Func<T>)` and `UseMonitoring<T>(string basePath, Func<T>)` (`T : ApplicationMonitor`) take a subclass factory; configure it with an object initializer, e.g. `() => new OperationsMonitor { CredentialValidator = ... }`.

`PageTitle` defaults to `"Monitoring"`.

## Access control

- The Basic Auth handler is attached to every dashboard route, including the JSON `/data` endpoints. Missing, malformed, or rejected credentials get `401` with a `WWW-Authenticate: Basic` header.
- With no `CredentialValidator` and no `AuthenticateAccountAsync` override, every credential is rejected (secure default since 1.6.3).
- The handler splits `user:password` on the first `:` and calls `AuthenticateAccountAsync(userEmail, userPassword)` synchronously on each request, so keep the check fast and non-blocking. The default implementation forwards to `CredentialValidator`.
- Basic Auth sends reusable credentials on every request: serve the dashboard over HTTPS and restrict network exposure. Do not hardcode credentials in source.

## Lifecycle

- The builder registers an internal `HttpServerHandler` that starts health sampling in `OnServerStarted`, stops it in `OnServerStopping` (`server.Stop()`), and feeds HTTP telemetry from request open/close events.
- Routes are created when the server first starts (`Start` binds the router). That moment loads `Store` state into the resources captured so far and starts the periodic store flush, so capture everything and set `Store` before `Start`.
- Disposing the server does not dispose the monitor. `ApplicationMonitor.Dispose`/`DisposeAsync` stop the flush loop and health sampling and dispose a `Store` that implements `IDisposable`/`IAsyncDisposable`; they do not perform a final flush. To keep the latest state on shutdown, hold a reference to the monitor (capture it in the callback or factory), call `monitor.FlushStore()` or `await monitor.FlushStoreAsync()`, then dispose it.
- `FlushStore`/`FlushStoreAsync` return `false` when there is no store, the monitor is disposed, a flush is already running, or saving threw (exceptions are swallowed).
