---
name: sisk-cadente-setup
description: Use this skill for setting up Sisk.Cadente as a Sisk engine or standalone HTTP/1.1 server, including SSL examples.
---
# Sisk.Cadente - Setup

## Using as Sisk Engine

The recommended way to use Cadente is via `CadenteHttpServerEngine` from `Sisk.Cadente.CoreEngine`:

```csharp
using Sisk.Core.Http;
using Sisk.Cadente.CoreEngine;

using var host = HttpServer.CreateBuilder()
    .UseEngine<CadenteHttpServerEngine>()
    .UseListeningPort(5000)
    .Build();

await host.StartAsync();
```

**With SSL (local development certificate only):**
```csharp
using var host = HttpServer.CreateBuilder()
    .UseEngine<CadenteHttpServerEngine>()
    .UseSsl(certificate: CertificateHelper.CreateTrustedDevelopmentCertificate("localhost"))
    .Build();
```

**With advanced host configuration:**
```csharp
using var engine = new CadenteHttpServerEngine(host =>
{
    host.TimeoutManager.ClientReadTimeout  = TimeSpan.FromSeconds(60);
    host.TimeoutManager.ClientWriteTimeout = TimeSpan.FromSeconds(60);
    host.TimeoutManager.SslHandshakeTimeout = TimeSpan.FromSeconds(10);
});

using var sisk = HttpServer.CreateBuilder()
    .UseEngine(engine)
    .Build();
```

- `CadenteHttpServerEngine(Action<HttpHost>)` — the action is called for every `HttpHost` created (one per listening port).
- `IdleConnectionTimeout` on the engine defaults to 90 s; controls both read and write timeouts applied to all hosts.

---

## Standalone Usage

Use `HttpHost` + `HttpHostHandler` directly without Sisk, similar to `HttpListener`:

```csharp
using Sisk.Cadente;

using var host = new HttpHost(15000)
{
    Handler = new MyHandler()
};

host.Start();
Thread.Sleep(-1); // keep alive

class MyHandler : HttpHostHandler
{
    public override async Task OnContextCreatedAsync(HttpHost host, HttpHostContext context)
    {
        context.Response.StatusCode = 200;
        context.Response.StatusDescription = "OK";
        context.Response.Headers.Set(new HttpHeader("Content-Type", "text/plain"));

        // chunked — no need to set Content-Length
        await using var stream = await context.Response.GetResponseStreamAsync(chunked: true);
        await using var writer = new StreamWriter(stream, leaveOpen: true);
        await writer.WriteLineAsync("Hello, world!");
    }
}
```

The standalone example blocks only to keep the demo alive. In an application, tie host disposal to its shutdown mechanism. Do not install or trust development certificates, or expose a listener beyond loopback, as a side effect of copying an example.

**Constructors:**
```csharp
new HttpHost(int port)                // listens on 127.0.0.1:<port>
new HttpHost(IPEndPoint endpoint)     // full control over IP and port
```

---
