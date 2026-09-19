---
name: sisk-cadente-host
description: Use this skill for Sisk.Cadente HttpHost, HttpHostHandler, HttpHostContext, request, and response APIs.
---
# Sisk.Cadente - Host APIs

## HttpHost

| Member | Description |
|---|---|
| `Handler` | `HttpHostHandler` instance. Set before `Start()`. |
| `HttpsOptions` | Assign an `HttpsOptions` to enable TLS. `null` = plain HTTP. |
| `TimeoutManager` | `HttpHostTimeoutManager` — per-host timeout settings. |
| `Endpoint` | The `IPEndPoint` the host listens on (read-only). |
| `IsDisposed` | Whether the host has been disposed. |
| `ServerNameHeader` | Static. Default `"Sisk"`. Sets the `Server` response header. |
| `Start()` | Begins accepting connections. Call once. |
| `Dispose()` | Stops accepting and releases resources. |

---

## HttpHostHandler

Override virtual methods to hook into the connection lifecycle:

```csharp
public class AppHandler : HttpHostHandler
{
    // Called for each HTTP request (one per message on a keep-alive connection)
    public override async Task OnContextCreatedAsync(HttpHost host, HttpHostContext context)
    {
        // read request, write response here
    }

    // Called once when a TCP client connects
    public override Task OnClientConnectedAsync(HttpHost host, HttpHostClient client)
    {
        client.State = new ClientSession(); // attach arbitrary state
        return Task.CompletedTask;
    }

    // Called once when the TCP client disconnects
    public override Task OnClientDisconnectedAsync(HttpHost host, HttpHostClient client)
    {
        (client.State as ClientSession)?.Dispose();
        return Task.CompletedTask;
    }
}
```

- `OnContextCreatedAsync` is called for **every request** on the connection (HTTP/1.1 keep-alive reuses TCP).
- `OnClientConnectedAsync` / `OnClientDisconnectedAsync` bracket the full TCP connection lifetime.
- `client.State` is a free `object?` slot — useful for associating per-connection data (sessions, auth state, etc.).

---

## HttpHostContext

Represents a single HTTP request–response exchange:

```csharp
public override async Task OnContextCreatedAsync(HttpHost host, HttpHostContext context)
{
    HttpHostContext.HttpRequest  req = context.Request;
    HttpHostContext.HttpResponse res = context.Response;
    HttpHostClient client            = context.Client;

    context.KeepAlive = true;  // default true; set false to close after this response

    // abort the TCP connection immediately without sending a response
    context.Abort();
}
```

---

## HttpHostContext.HttpRequest

```csharp
string method      = req.Method;         // "GET", "POST", etc.
string path        = req.Path;           // "/api/users?id=1"
long   length      = req.ContentLength;
bool   hasBody     = req.HasBody;        // true if ContentLength > 0 or chunked

HttpHeaderList headers = req.Headers;    // read-only

// reading the body
Stream bodyStream = req.GetRequestStream();
using var reader  = new StreamReader(bodyStream);
string body       = await reader.ReadToEndAsync();
```

- `GetRequestStream()` handles both fixed-length and chunked bodies transparently.
- `HasBody` is `true` for chunked requests even if the actual body is empty — always check before reading.
- Automatically sends `100 Continue` if the client sent `Expect: 100-continue`.

---

## HttpHostContext.HttpResponse

```csharp
res.StatusCode        = 200;
res.StatusDescription = "OK";
res.Headers.Set(new HttpHeader("Content-Type", "application/json"));
byte[] bodyBytes = Encoding.UTF8.GetBytes(body);
res.Headers.Set(new HttpHeader("Content-Length", bodyBytes.Length.ToString()));

await using var stream = await res.GetResponseStreamAsync(); // fixed-length
await stream.WriteAsync(bodyBytes);
```

**Chunked response (no Content-Length needed):**
```csharp
res.Headers.Set(new HttpHeader("Content-Type", "text/plain"));

await using var stream = await res.GetResponseStreamAsync(chunked: true);
await using var writer = new StreamWriter(stream, leaveOpen: true);
await writer.WriteLineAsync("chunk 1");
await writer.WriteLineAsync("chunk 2");
```

- `GetResponseStreamAsync(chunked: false)` — **requires** `Content-Length` in encoded bytes to be set beforehand, or throws `InvalidOperationException`. Do not use string length for UTF-8 bodies.
- `GetResponseStreamAsync(chunked: true)` — sets `Transfer-Encoding: chunked` automatically and removes `Content-Length`.
- Headers are sent on the first call to `GetResponseStreamAsync`; adding headers after that has no effect.
- Default headers added automatically: `Date`, `Server`.

---
