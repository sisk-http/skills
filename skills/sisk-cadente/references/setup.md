# Setup and integration

For additional details, see the [official Sisk Cadente documentation](https://docs.sisk-framework.org/docs/cadente) and [Working with SSL](https://docs.sisk-framework.org/docs/ssl).

## Use Cadente through Sisk

Install `Sisk.Cadente.CoreEngine` (`dotnet add package Sisk.Cadente.CoreEngine`), then configure the Sisk server as usual:

```csharp
using Sisk.Cadente.CoreEngine;
using Sisk.Core.Http;

using var server = HttpServer.CreateBuilder()
    .UseEngine(new CadenteHttpServerEngine())
    .UseListeningPort("http://127.0.0.1:5000/")
    .Build();

await server.StartAsync();
```

`UseEngine<CadenteHttpServerEngine>()` is the equivalent form when no setup action is needed.

Pass a host setup action only when configuration must apply to every Cadente `HttpHost` created for the Sisk listening hosts. It runs after the engine's own defaults, so it overrides them:

```csharp
using var engine = new CadenteHttpServerEngine(host => {
    host.TimeoutManager.HeaderParsingTimeout = TimeSpan.FromSeconds(10);
    host.TimeoutManager.BodyDrainTimeout = TimeSpan.FromSeconds(10);
});

using var server = HttpServer.CreateBuilder()
    .UseEngine(engine)
    .UseListeningPort("http://127.0.0.1:5000/")
    .Build();
```

Each host's `ClientReadTimeout` and `ClientWriteTimeout` are set from the engine's `IdleConnectionTimeout`. When the engine runs inside an `HttpServer`, that value is overwritten at startup by `HttpServerConfiguration.IdleConnectionTimeout` (default 120 seconds), so change it with `UseConfiguration(c => c.IdleConnectionTimeout = ...)`, or override the host timeouts in the setup action. Setting `CadenteHttpServerEngine.IdleConnectionTimeout` directly has no effect on a server built this way.

Keep application code on Sisk's request/response abstractions with this integration. `HttpRequest.DisconnectToken` is backed by Cadente's per-connection disconnect token.

## TLS and mTLS

For local development, the parameterless `UseSsl()` creates and trusts a reusable certificate for `localhost` and `127.0.0.1` through `CertificateHelper.CreateTrustedDevelopmentCertificate`:

```csharp
using var server = HttpServer.CreateBuilder()
    .UseEngine<CadenteHttpServerEngine>()
    .UseListeningPort("https://127.0.0.1:5001/")
    .UseSsl()
    .Build();
```

This helper writes a `.pfx` under the current user's local application data (`.sisk/development-certs`) and installs the certificate in the current user's Root store. Use it only for local development. `CertificateHelper` (namespace `Sisk.Core.Helpers`) also offers `CreateDevelopmentCertificate(params string[] dnsNames)`, which creates a self-signed certificate without installing trust and requires at least one DNS name. Dispose returned certificates. `UseSsl(X509Certificate2)` takes an explicit certificate with default options.

For mTLS or deployed certificates, pass explicit `ListeningHostSslOptions` (namespace `Sisk.Core.Http`):

```csharp
using var server = HttpServer.CreateBuilder()
    .UseEngine<CadenteHttpServerEngine>()
    .UseListeningPort("https://127.0.0.1:5001/")
    .UseSsl(new ListeningHostSslOptions(serverCertificate) {
        ClientCertificateRequired = true,
        CheckCertificateRevocation = true
    })
    .Build();
```

The engine copies `ServerCertificate`, `AllowedProtocols`, `ClientCertificateRequired`, and `CheckCertificateRevocation` into each host's `HttpsOptions`. Defaults are TLS 1.2 | 1.3, client certificate not required, and **revocation checking off**, so set `CheckCertificateRevocation = true` explicitly for mTLS and do not disable it in production examples. Revocation checking uses the online mode, so the server needs network access to the issuer's revocation endpoints. Validate the full TLS and client-certificate chain in the target environment. Sisk's `HttpRequest` does not expose the client certificate; read `HttpHostClient.ClientCertificate` in a standalone handler when the identity is needed.

On a listener with TLS enabled, Cadente first peeks at the connection:

- Plain HTTP is answered with `301 Moved Permanently` to `https://{Host header}{path}` (or `400` when there is no `Host`), and `HttpServer.ListeningPrefixes` reports the `https` scheme.
- A failed TLS handshake is answered with a plain `400`, and the connection is closed.

## Standalone is low-level only

Use `Sisk.Cadente.HttpHost` directly only when Sisk's routing, middleware, and server abstractions are intentionally not needed:

```csharp
using Sisk.Cadente;

using var host = new HttpHost(15000) {
    Handler = new AppHandler()
};

host.Start();
await Task.Delay(Timeout.InfiniteTimeSpan);
```

`HttpHost(int)` listens on the loopback endpoint for that port; use `HttpHost(IPEndPoint)` when the bind address must be explicit. Set `Handler`, `HttpsOptions`, and timeouts before `Start()`. `Stop()` closes the listener and `Start()` may be called again; `Dispose()` is final. See [Host APIs](host-apis.md) for the handler contract and response framing rules.
