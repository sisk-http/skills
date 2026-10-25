---
name: sisk-monitoring
description: Use when adding, securing, customizing, or diagnosing the Sisk.Monitoring dashboard - UseMonitoring registration, Counter, Meter, LogStream capture, Basic Auth, persistence, health and HTTP telemetry.
---

# Sisk.Monitoring

For additional details, see the [official Sisk documentation](https://docs.sisk-framework.org/docs/) and the [official Sisk.Monitoring package page](https://www.nuget.org/packages/Sisk.Monitoring).

`Sisk.Monitoring` embeds a Basic Auth dashboard in a Sisk server (`UseMonitoring` on `HttpServerHostContextBuilder`) showing HTTP telemetry, `Counter`, `Meter`, `LogStream` and server-health data.

Invariants that cause most failures:

- Every dashboard route answers `401` until `ApplicationMonitor.CredentialValidator` is set or `AuthenticateAccountAsync` is overridden.
- One `ApplicationMonitor` serves one route prefix; reusing an instance for a second prefix throws `InvalidOperationException`.
- Capture resources and set `Store`/`StoreFlushInterval` before the server starts (normally inside the `UseMonitoring` callback or factory). State is restored and the flush loop starts when the server first starts.
- `UseMonitoring` never disposes the monitor and nothing flushes the store on shutdown; see the lifecycle section of the setup reference.

Read the reference that matches the task:

- [setup-and-access](./references/setup-and-access.md) - read when registering the dashboard, choosing a `UseMonitoring` overload, configuring credentials, or handling start/stop/disposal.
- [resources-and-persistence](./references/resources-and-persistence.md) - read when adding or changing `Counter`, `Meter`, `LogStream` capture, labels/groups/keys/pinning, the `Store`, health history, or HTTP telemetry.
- [routes-and-customization](./references/routes-and-customization.md) - read when calling dashboard routes, protecting or proxying them, or overriding pages and JSON endpoints in an `ApplicationMonitor` subclass.
