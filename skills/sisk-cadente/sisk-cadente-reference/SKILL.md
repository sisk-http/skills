---
name: sisk-cadente-reference
description: Use this skill for Sisk.Cadente HttpHostClient, HttpHeaderList, HttpsOptions, timeout manager, and do/don't guidance.
---
# Sisk.Cadente - Reference Types

## HttpHostClient

```csharp
IPEndPoint         ep      = client.ClientEndpoint;    // remote IP:port
X509Certificate?   cert    = client.ClientCertificate; // null unless mTLS configured
CancellationToken  dc      = client.DisconnectToken;   // raised on disconnection
object?            state   = client.State;             // free slot for connection-scoped data
```

---

## HttpHeaderList

Used for both request (read-only) and response (read-write) headers:

```csharp
// add without replacing
headers.Add(new HttpHeader("X-Custom", "value"));

// add or replace (by name, case-insensitive)
headers.Set(new HttpHeader("Content-Type", "application/json"));

// remove all with a given name
headers.Remove(HttpHeaderName.ContentLength);

// check existence
bool has = headers.Contains(HttpHeaderName.ContentType);

// get value(s)
string value = headers.Get(HttpHeaderName.ContentType); // first match or ""
```

Use `HttpHeaderName` constants (`HttpHeaderName.ContentType`, `HttpHeaderName.ContentLength`, `HttpHeaderName.TransferEncoding`, etc.) instead of raw strings to avoid typos.

---

## HttpsOptions

Enables TLS for an `HttpHost`:

```csharp
var cert = X509Certificate2.CreateFromPemFile("cert.pem", "key.pem");

host.HttpsOptions = new HttpsOptions(cert)
{
    AllowedProtocols          = SslProtocols.Tls12 | SslProtocols.Tls13, // default
    ClientCertificateRequired = false,                                     // mTLS
    CheckCertificateRevocation = false
};
```

---

## HttpHostTimeoutManager

Configure per-host timeouts:

```csharp
host.TimeoutManager.ClientReadTimeout    = TimeSpan.FromSeconds(30); // default
host.TimeoutManager.ClientWriteTimeout   = TimeSpan.FromSeconds(30); // default
host.TimeoutManager.SslHandshakeTimeout  = TimeSpan.FromSeconds(5);  // default
host.TimeoutManager.HeaderParsingTimeout = TimeSpan.FromSeconds(30); // default
host.TimeoutManager.BodyDrainTimeout     = TimeSpan.FromSeconds(30); // default
```

---

## Do's and Don'ts

- **Do** set `Content-Length` before calling `GetResponseStreamAsync()` for non-chunked responses.
- **Do** use `chunked: true` when the response size is not known upfront (streaming, SSE-like flows).
- **Do** use `context.KeepAlive = false` when you want to close the connection after the response (e.g., on protocol errors).
- **Do** use `client.State` in `OnClientConnectedAsync` to attach session/auth state and clean it up in `OnClientDisconnectedAsync`.
- **Don't** set headers after `GetResponseStreamAsync()` is called — they won't be sent.
- **Do** keep the chunked stream alive until all writes complete, then dispose it with `await using` to flush the terminal chunk. When wrapping it in a writer, dispose the writer first; `leaveOpen: true` lets the outer scope own the stream.
- **Don't** share `HttpHostContext` instances across threads or requests.
