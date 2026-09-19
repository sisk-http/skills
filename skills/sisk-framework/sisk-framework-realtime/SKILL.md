---
name: sisk-framework-realtime
description: Use this skill for Sisk Framework WebSockets, Server-Sent Events, and LogStream logging.
---
# Sisk Framework - Realtime and Logging

## Web Sockets

```csharp
router.MapGet("/ws", async req =>
{
    using var ws = await req.GetWebSocketAsync();

    while (await ws.ReceiveMessageAsync(timeout: TimeSpan.FromSeconds(30)) is { } msg)
    {
        string text = msg.GetString();
        await ws.SendAsync($"Echo: {text}");
    }

    return await ws.CloseAsync();
});
```

- `ReceiveMessageAsync` returns `null` on timeout, cancellation, or disconnection.
- It's not possible to read and write simultaneously.

**Ping policy (keep connection alive):**
```csharp
ws.PingPolicy.Start(
    dataMessage: "ping",
    interval: TimeSpan.FromSeconds(10));
```

---

## Server-Sent Events (SSE)

```csharp
router.MapGet("/events", async req =>
{
    using var sse = await req.GetEventSourceAsync();

    sse.AppendHeader("X-Custom", "value"); // must come before any Send

    foreach (var item in GetItems())
    {
        await sse.SendAsync(item.ToString());
        await Task.Delay(500);
    }

    return sse.Close();
});
```

**Persistent identified connection (WaitForFail):**
```csharp
router.MapGet("/live", req =>
{
    using var sse = req.GetEventSource("client-" + req.Query["id"]);
    sse.WithPing(p => { p.DataMessage = "ping"; p.Interval = TimeSpan.FromSeconds(5); p.Start(); });
    sse.WaitForFail(TimeSpan.FromMinutes(10));
    return sse.Close();
});

// from elsewhere, send to specific connection
HttpRequestEventSource? conn = server.EventSources.GetByIdentifier("client-42");
conn?.Send("New event!");

// broadcast
foreach (var e in server.EventSources.Find(id => id.StartsWith("client-")))
    e.Send("Broadcast!");
```

- Browser `EventSource` reconnects automatically after server closure. A termination event only stops reconnection if the client handles it by calling `EventSource.close()`.
- Native browser `EventSource` uses GET and does not accept arbitrary request headers; distinguish it from fetch-based SSE clients.
- `server.EventSources.All` lists all active identified connections.

---

## Logging (LogStream)

```csharp
// File
config.AccessLogsStream = new LogStream("logs/access.log");
config.ErrorsLogsStream = new LogStream("logs/error.log");
config.ThrowExceptions  = false; // necessary for errors to go to ErrorsLogsStream

// Console
config.AccessLogsStream = new LogStream(Console.Out);

// Custom application log
var appLog = new LogStream("logs/app.log");
appLog.WriteLine("Server started at {0}", DateTime.Now);
```

**Custom format:**
```csharp
config.AccessLogsFormat = "%dd/%dmm/%dy %tH:%ti:%ts %tz %ls %ri %rm %rs://%ra%rz%rq [%sc %sd] %lin -> %lou in %lmsms";
// Uses HttpServerConfiguration.DefaultAccessLogFormat for default
```

Useful variables: `%ri` (client IP), `%rm` (method), `%rz` (path), `%sc` (status code), `%lms` (ms), `%{header-name}` (request header), `%{:header-name}` (response header).

**Automatic rotation:**
```csharp
new LogStream("logs/access.log")
    .ConfigureRotatingPolicy(
        maximumSize: 64 * SizeHelper.UnitMb,
        dueTime: TimeSpan.FromHours(6));
```

**Custom LogStream:**
```csharp
public class PrefixedLogStream : LogStream
{
    protected override void WriteLineInternal(string line)
        => base.WriteLineInternal($"[{DateTime.Now:g}] {line}");
}
```

- `LogStream` implements `IAsyncDisposable`; ensures pending log writes on dispose.
- Check the installed version's directory-creation behavior and ensure the log path is writable before enabling file logging.
- Rotation works only on file-based LogStreams.

---
