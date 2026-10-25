# LogStream

For additional details, see the official Sisk documentation on [logging](https://docs.sisk-framework.org/docs/features/logging) and the [`LogStream` API reference](https://docs.sisk-framework.org/api/Sisk.Core.Http.LogStream.html).

Configure access and error logs on `HttpServerConfiguration`:

```csharp
configuration.AccessLogsStream = new LogStream("logs/access.log");
configuration.ErrorsLogsStream = new LogStream("logs/error.log");
configuration.ThrowExceptions = false;
```

`LogStream` supports console, stream, and file output. Application-specific logs can use independent instances:

```csharp
await using var applicationLog = new LogStream("logs/application.log");
applicationLog.WriteLine("Worker started");
```

Writes are queued and consumed in the background. `Dispose`/`DisposeAsync` cancel the consumer while closing, so entries still queued may be lost: call `await stream.FlushAsync()` (or `Flush()`) before disposing application-owned streams at shutdown. File-based streams can use `ConfigureRotatingPolicy(maximumSize, dueTime)`.

Since 1.7, static `LogStream.SafeWriteToFile` and `SafeAppendToFile` (plus `Async` variants) write a file and return `false` instead of throwing on I/O failure; calls for the same path are serialized within the process.
