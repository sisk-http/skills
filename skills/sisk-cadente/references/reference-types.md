# Reference types

For additional details, see the [official Sisk.Cadente API documentation](https://docs.sisk-framework.org/api/Sisk.Cadente.html).

## `HttpHeaderList` and `HttpHeader`

`HttpHeaderList` implements `IList<HttpHeader>`. `context.Request.Headers` is read-only (mutating it throws `InvalidOperationException`); `context.Response.Headers` is mutable and starts with `Date` and `Server`. Names are matched case-insensitively.

`HttpHeaderName` is internal to `Sisk.Cadente`, so standalone code uses string literals. (`Sisk.Core.Http.HttpKnownHeaderNames` exists only when the `Sisk.HttpServer` package is referenced.)

```csharp
HttpHeaderList headers = context.Request.Headers;

bool hasContentType = headers.Contains("Content-Type");
string? contentType = headers.Get("Content-Type").FirstOrDefault();

context.Response.Headers.Set(new HttpHeader("Content-Type", "application/json"));
context.Response.Headers.Remove("Content-Length");
```

`Get(string)` returns `string[]`, with an empty array when no header matches. Use `FirstOrDefault()` only when one value is intended; preserve or process all values for repeated headers. `Set` replaces every header with the same name, while `Add` preserves existing values.

`new HttpHeader(name, value)` trims the value and throws `ArgumentException` for an empty name, a name containing token-forbidden characters (such as spaces, `:`, `/`, `()<>@,;\"[]?={}`), or a value containing CR, LF, or NUL. Do not catch this to "sanitize" untrusted input; reject or encode it before building the header.

## `HttpHostClient`

```csharp
IPEndPoint endpoint = client.ClientEndpoint;
X509Certificate? certificate = client.ClientCertificate;
bool isTls = client.IsSecureConnection;
CancellationToken disconnected = client.DisconnectToken;
object? state = client.State;
```

`ClientCertificate` is the TLS peer certificate and is non-null only when the client presented one (`HttpsOptions.ClientCertificateRequired` forces it). `State` is the only settable member, scoped to the connection. `DisconnectToken` is canceled when the connection ends.

Use `client.DisconnectToken` in a standalone `HttpHostHandler` to stop work for a disconnected client. When Cadente runs through `CadenteHttpServerEngine`, Sisk code uses `HttpRequest.DisconnectToken`, which is backed by the same token. Code meant to work with other Sisk engines must treat it as optional, because the default `HttpListener` engine returns a non-canceling token.

## `HttpsOptions`

Assign `HttpsOptions` to a standalone host before starting it:

```csharp
host.HttpsOptions = new HttpsOptions(serverCertificate) {
    AllowedProtocols = SslProtocols.Tls12 | SslProtocols.Tls13,
    ClientCertificateRequired = true,
    CheckCertificateRevocation = true
};
```

Defaults: `AllowedProtocols` is TLS 1.2 | TLS 1.3, `ClientCertificateRequired` is `false`, and `CheckCertificateRevocation` is `false` (when enabled, revocation is checked online). `ServerCertificate` is read-only after construction.

Enable `ClientCertificateRequired` only for mTLS endpoints that validate client identities, and set `CheckCertificateRevocation = true` explicitly there, since it is off by default. Validate the chosen policy in the deployment environment. For ordinary Sisk integration, configure TLS through `ListeningHostSslOptions` (see [Setup and integration](setup.md)) instead of creating `HttpsOptions` in application handlers.

## `HttpHostTimeoutManager`

Each standalone host exposes `TimeoutManager`; the type cannot be constructed directly. Defaults are shown below:

```csharp
host.TimeoutManager.ClientReadTimeout = TimeSpan.FromSeconds(30);
host.TimeoutManager.ClientWriteTimeout = TimeSpan.FromSeconds(30);
host.TimeoutManager.SslHandshakeTimeout = TimeSpan.FromSeconds(5);
host.TimeoutManager.HeaderParsingTimeout = TimeSpan.FromSeconds(30);
host.TimeoutManager.BodyDrainTimeout = TimeSpan.FromSeconds(30);
```

Values are read when each connection starts, so set them before `Start()`. `SslHandshakeTimeout` also bounds the initial TLS-or-plain-HTTP probe. `BodyDrainTimeout` bounds Cadente's attempt to consume an unread request body before an HTTP/1.1 connection is reused; tune it with expected payload sizes and network conditions, and set `context.KeepAlive = false` if a request must not be drained.
