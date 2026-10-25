# Routes and Customization

For additional details, see the [official Sisk documentation](https://docs.sisk-framework.org/docs/) and the [official Sisk.Monitoring package page](https://www.nuget.org/packages/Sisk.Monitoring).

## Routes

All routes are `GET`, sit under the base path (`/monitoring` by default), require Basic Auth, and log only on errors (`LogOutput.ErrorLog`).

| Path | Purpose |
| --- | --- |
| `{prefix}/` | Overview: HTTP telemetry plus pinned resources. |
| `{prefix}/data?period=1h\|1d\|7d` | Overview JSON: HTTP metrics and pinned counters. |
| `{prefix}/counters` | Counters page. |
| `{prefix}/counters/data` | Counter values as JSON. |
| `{prefix}/meters` | Meter charts page. |
| `{prefix}/meters/data[?name=]` | Meter series as JSON; `name` filters by `Label` or `SanitizedLabel`. |
| `{prefix}/logstream/{name}` | Log-stream viewer page (static shell; lines load from `/data`). |
| `{prefix}/logstream/{name}/data` | Buffered log lines as JSON, oldest first. |
| `{prefix}/logstream/{name}/download` | Log file download; `404` without a backing file. |
| `{prefix}/health` | Server-health page. |
| `{prefix}/health/data?period=1h\|1d\|7d\|30d` | Aggregated health history as JSON. |

`{name}` is matched case-insensitively against the definition `Label`; an unknown name returns `404`. The dashboard's own routes are not counted in HTTP telemetry.

When fronting the dashboard with a reverse proxy, forward the whole prefix and the `Authorization` header, and terminate TLS in front of it.

## Customizing

Prefer `PageTitle`, `Group`, `DashboardPinned` and `Key` before overriding anything. To change pages or payloads, subclass `ApplicationMonitor` and register it with `UseMonitoring(() => new MyMonitor())`. The route set is fixed; only these protected virtual members are extension points:

| Member | Serves |
| --- | --- |
| `AuthenticateAccountAsync(string userEmail, string userPassword)` -> `ValueTask<bool>` | Credential check. |
| `WriteSidebar(string? activeNavItem)` -> `HtmlElement` | Sidebar `nav`. |
| `GetDashboardPageHtmlAsync`, `GetCountersPageHtmlAsync`, `GetMetersPageHtmlAsync`, `GetServerHealthPageHtmlAsync`, `GetLogStreamPageHtmlAsync(request, logStream)` | HTML pages. |
| `GetDashboardDataAsync`, `GetCountersDataAsync`, `GetMetersDataAsync`, `GetServerHealthDataAsync`, `GetLogStreamDataAsync(request, logStream)` | JSON payloads. |
| `GetLogStreamFileDownloadAsync(request, logStream)` | Log file download. |

Page and data methods take an `HttpRequest` and return `ValueTask<HttpResponse>`. The built-in pages read their JSON endpoints through `data-*` attributes, so keep payload shapes and route paths intact when overriding only one side. HTML is built with TinyComponents (`using TinyComponents;`).

`activeNavItem` is `"dashboard"`, `"health"`, `"counters"`, `"meters"`, or a log stream's `SanitizedLabel`.

```csharp
using Sisk.Monitoring;
using TinyComponents;

sealed class OperationsMonitor : ApplicationMonitor {
    protected override HtmlElement WriteSidebar(string? activeNavItem) {
        var sidebar = base.WriteSidebar(activeNavItem);
        sidebar += new HtmlElement("div", "Operations runbook: see the internal wiki");
        return sidebar;
    }

    protected override ValueTask<bool> AuthenticateAccountAsync(string userEmail, string userPassword) =>
        MyDirectory.ValidateAsync(userEmail, userPassword);
}
```

Overridden pages and payloads are served to every authenticated user: do not put secrets, tokens or personal data in labels, counters, meters, log lines or custom markup.
