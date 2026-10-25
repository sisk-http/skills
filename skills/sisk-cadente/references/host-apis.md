# Host APIs

For additional details, see the [official Sisk.Cadente API documentation](https://docs.sisk-framework.org/api/Sisk.Cadente.html).

Use these APIs only for deliberate standalone `HttpHost` work. For normal Sisk applications, configure `CadenteHttpServerEngine` and use Sisk request handlers instead.

## Host and handler lifecycle

`HttpHost` owns one listener. Set its `Handler` before `Start()`; without a handler, accepted connections are closed immediately. `HttpsOptions`, `TimeoutManager`, and the read-only `Endpoint` describe its transport configuration.

```csharp
using System.Text;
using Sisk.Cadente;

sealed class AppHandler : HttpHostHandler {
    public override async Task OnContextCreatedAsync(HttpHost host, HttpHostContext context) {
        byte[] body = Encoding.UTF8.GetBytes("Hello, world!");

        context.Response.StatusCode = 200;
        context.Response.Headers.Set(new HttpHeader("Content-Type", "text/plain; charset=utf-8"));
        context.Response.Headers.Set(new HttpHeader("Content-Length", body.Length.ToString()));

        await context.Response.WriteInlineContentAsync(body);
    }
}
```

`OnContextCreatedAsync` runs once per request, including requests sharing a keep-alive connection. `OnClientConnectedAsync` and `OnClientDisconnectedAsync` run once per TCP client, after the TLS handshake; use `HttpHostClient.State` for connection-scoped data. All three are virtual with no-op defaults.

- A handler that returns without opening a response stream still gets its status and headers sent, with `Content-Length: 0`.
- An exception that escapes `OnContextCreatedAsync` closes the connection without any response; catch errors and write a `500` yourself.
- Cadente adds `Date` and `Server` headers to every response. The static `HttpHost.ServerNameHeader` (default `Sisk`) sets the `Server` value.

## Request handling and connection reuse

`context.Request` exposes `Method`, `Path`, `ContentLength`, `HasBody`, `Headers`, and `GetRequestStream()`.

- `Path` is the raw request target and includes the query string.
- `ContentLength` is `-1` for chunked requests, and `HasBody` can be true for a chunked request whose actual body is empty.
- `GetRequestStream()` returns `Stream.Null` when `Content-Length` is 0, decodes chunked bodies transparently, and sends `100 Continue` first when the client sent `Expect: 100-continue`. A malformed chunked body throws `Sisk.Cadente.Streams.ChunkParseException` (an `IOException`).

Before reusing a keep-alive connection, consume the request body or opt out of reuse:

```csharp
if (context.Request.HasBody) {
    await context.Request.GetRequestStream().CopyToAsync(Stream.Null);
}

// Use this instead when the body should not be read or drained.
context.KeepAlive = false;
```

Cadente also attempts to drain an unread body before reuse, bounded by `BodyDrainTimeout`; if the drain times out, or an `Expect: 100-continue` body was never read, it closes the connection instead. Explicitly consume bodies that the handler accepts, and set `KeepAlive = false` for early rejection or when reuse is inappropriate.

`context.Abort()` closes the connection stream; Cadente generates no response. Do not share a `HttpHostContext` across requests or retain it after the handler completes.

## Request validation

Before the handler runs, Cadente parses and validates the request line and headers (HTTP/1.0 and 1.1 request lines are accepted, header block up to 8 KB). Pipelined requests on keep-alive connections are supported.

- A header block larger than 8 KB gets `431 Request Header Fields Too Large` with `Connection: close`.
- Other malformed requests close the connection and **never reach the handler**. Rejected cases include invalid method tokens; request targets with a `#` fragment or bytes outside `0x21-0x7E`; a missing `Host` on an HTTP/1.1 request, or an empty or duplicate `Host`; a `Transfer-Encoding` other than a single `chunked`, or duplicated; `Transfer-Encoding` together with `Content-Length`; invalid or conflicting `Content-Length`; and obsolete line folding, bare LF, or header names/values with forbidden characters.
- Header-read timeouts (`HeaderParsingTimeout`) also close the connection silently.

Do not add handler code to re-validate these cases; test them at the socket level if a behavior must be proven.

## Response framing

Set the status and every response header **before** opening the response stream: the headers are captured at that point (sent immediately for chunked, buffered with the first write otherwise), and opening a second stream or writing inline content afterwards throws `InvalidOperationException`. Choose exactly one body framing strategy:

- **Known length:** set `Content-Length`, then call `WriteInlineContentAsync(ReadOnlyMemory<byte>)` (or the synchronous `WriteInlineContent(ReadOnlySpan<byte>)`) for an in-memory body, or `GetResponseStreamAsync(chunked: false)` to write it yourself.
- **Unknown or streaming length:** call `GetResponseStreamAsync(chunked: true)` and do not set `Content-Length`. Cadente sets `Transfer-Encoding: chunked` and removes `Content-Length`.

```csharp
// Known length, streamed by the handler.
byte[] body = Encoding.UTF8.GetBytes(json);
context.Response.Headers.Set(new HttpHeader("Content-Length", body.Length.ToString()));
await using Stream output = await context.Response.GetResponseStreamAsync(chunked: false);
await output.WriteAsync(body);
```

```csharp
// Streaming: no Content-Length.
context.Response.Headers.Set(new HttpHeader("Content-Type", "text/plain"));
await using Stream output = await context.Response.GetResponseStreamAsync(chunked: true);
await output.WriteAsync(Encoding.UTF8.GetBytes("first chunk\n"));
```

A non-chunked response without `Content-Length` throws `InvalidOperationException`, except for a `101` response whose `Upgrade` header contains `websocket`. Disposing the chunked stream writes the terminal chunk.
