# Realtime

For additional details, see the official Sisk documentation on [WebSockets](https://docs.sisk-framework.org/docs/features/websockets) and [server-sent events](https://docs.sisk-framework.org/docs/features/server-sent-events).

## WebSockets

Acquire a socket once per request and dispose it even on cancellation or protocol errors.

```csharp
router.MapGet("/ws", async request =>
{
    using HttpWebSocket socket = await request.GetWebSocketAsync(identifier: "client-42");

    while (await socket.ReceiveMessageAsync(request.DisconnectToken) is { } message)
        await socket.SendAsync(message.GetString(), request.DisconnectToken);

    return await socket.CloseAsync();
});
```

Keep the owning `HttpServer` instance when other application code must publish outside the route callback:

```csharp
if (server.WebSockets.GetByIdentifier("client-42") is { IsClosed: false } socket)
    await socket.SendAsync("order-ready", cancellationToken);
```

Only identified sockets are registered. A new socket with the same identifier closes and replaces the previous one. `server.WebSockets.Find(...)` and `All()` return active connections.

`ReceiveMessageAsync` returns `null` on timeout, cancellation, or disconnect. Close and dispose the socket to unregister it and release its ping policy and semaphores. Received messages are buffered; their maximum size follows `MaximumContentLength` when it is positive, otherwise the runtime array maximum. A message above the limit closes the connection with `MessageTooBig`. Keep sends and receives serialized by application logic; do not assume an unbounded message stream.

## Server-Sent Events

SSE, response streams, and WebSockets are mutually exclusive request streaming modes. Dispose or close the event source on every exit path; disposal unregisters it, clears queued messages, stops pinging, and wakes `WaitForFailAsync`.

```csharp
router.MapGet("/events", async request =>
{
    using HttpRequestEventSource events = await request.GetEventSourceAsync("client-42");
    events.AppendHeader("Cache-Control", "no-cache");

    await events.SendAsync("connected");
    await events.WaitForFailAsync(request.DisconnectToken);
    return await events.CloseAsync();
});
```

The identifier also allows publishing outside the request callback:

```csharp
if (server.EventSources.GetByIdentifier("client-42") is { IsActive: true } events)
    await events.SendAsync("order-ready");
```

Only identified event sources are registered. A new event source with the same identifier closes and replaces the previous one. `server.EventSources.Find(...)` and `All()` return active connections.

Set headers before sending data. `SendAsync` returns `false` when inactive; check it in long-lived producers. Use `WaitForFailAsync` with a cancellation token or bounded timeout instead of obsolete `KeepAlive`. `Cancel()` drops queued unsent messages but does not close the connection. Browsers reconnect SSE automatically, so define an application termination event when reconnects should stop.
