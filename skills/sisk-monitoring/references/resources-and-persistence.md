# Resources and Persistence

For additional details, see the [official Sisk documentation](https://docs.sisk-framework.org/docs/) and the [official Sisk.Monitoring package page](https://www.nuget.org/packages/Sisk.Monitoring).

## `MonitoringDefinition<T>`

Wraps a `Counter`, `Meter` or `LogStream` for `CaptureCounter`, `CaptureMeter` or `CaptureLogStream`. Constructors: `(label, instance)` and `(label, group, instance)`; a null/whitespace label throws.

```csharp
new MonitoringDefinition<Counter>("Database queries", "Database", databaseCounter) {
    DashboardPinned = true,
    Key = "db-queries"
};
```

- `Label` is the display name and, for log streams, the `{name}` in the route (matched case-insensitively).
- `Group` groups sidebar/page entries; null falls under "Ungrouped".
- `DashboardPinned` shows the resource on the overview page (counters, meters and log streams).
- `Key` is the persistence key; `StorageKey` is `Key`, or `Label` when `Key` is unset. Set `Key` before renaming a label, otherwise persisted state no longer matches.
- `SanitizedLabel` is `WebUtility.UrlEncode(Label)`; the dashboard builds links and `data-*` ids from it.

Keep the instances as singletons so values survive across requests.

## Counter

Thread-safe sum of increments that expire. `Current` sums only unexpired increments; `DefaultDuration` defaults to 1 hour.

```csharp
var counter = new Counter { DefaultDuration = TimeSpan.FromHours(6) };
counter.Increment();                                  // +1, expires after DefaultDuration
counter.Increment(5);
counter.Increment(1, TimeSpan.FromMinutes(30));       // custom expiry
var initial = new Counter(10, TimeSpan.FromHours(1)); // initial value + DefaultDuration
```

`Reset()` clears all increments. The dashboard counter value is a window total, not a lifetime total.

## Meter

Time series in 1-second buckets over a sliding window (default 24 hours; the constructor throws for a non-positive window). `Read()` aggregates with a 1-hour tick; `Read(TimeSpan tick)` throws for a non-positive tick and returns one `MeterReading(double Value, DateTime Timestamp)` per tick across the window, where `Timestamp` is the tick end in local time (`DateTime.Now`).

```csharp
var meter = new Meter(TimeSpan.FromHours(12));
meter.Increment();
meter.Increment(42.5);

foreach (var reading in meter.Read(TimeSpan.FromMinutes(15)))
    Console.WriteLine($"{reading.Timestamp}: {reading.Value}");
```

`Reset()` clears all readings. `Increment` is amortized O(1) for both `Counter` and `Meter` (1.7.0-rc3).

## LogStream capture

```csharp
var applicationLog = new LogStream("logs/application.log");
monitor.CaptureLogStream(
    new MonitoringDefinition<LogStream>("Application log", applicationLog),
    bufferLineCount: 500);
```

- `bufferLineCount` defaults to 500. `CaptureLogStream` calls `LogStream.StartBuffering` only if the stream is not already buffering.
- The page reads buffered lines (`PeekEntries()`), oldest first. When the buffer is empty and the stream has a `FilePath`, the data route tails that many lines from the file.
- The download route serves `LogStream.FilePath`; it returns `404` when the stream has no file or the file does not exist.
- The buffer is in-memory only and not persisted by the `Store`. It is not rotation or retention; use `LogStream.ConfigureRotatingPolicy` for file rotation.

## Persistence (`Store`)

`ApplicationMonitor.Store` is an `IMonitoringStore` (`SaveAsync(MonitoringSnapshot, ct)`, `LoadAsync(ct)`). `FileMonitoringStore(string filePath)` is the built-in JSON file implementation: it writes a temporary file, replaces the target, and `LoadAsync` returns `null` for a missing, unreadable or invalid file.

```csharp
monitor.Store = new FileMonitoringStore("data/monitoring.json");
monitor.StoreFlushInterval = TimeSpan.FromSeconds(30); // default 10 s; must be > 0
```

- Persisted: counter increments (plus each counter's `DefaultDuration`), meter buckets, and health history. Not persisted: log-stream buffers and HTTP traffic telemetry.
- On first start, entries are matched to captured resources by `StorageKey` (case-insensitive). Restored counters take the persisted `DefaultDuration`; expired counter increments and meter buckets outside the window are dropped; health samples older than 30 days are dropped.
- A flush runs every `StoreFlushInterval`, and `FlushStore()`/`FlushStoreAsync()` flush on demand. Failures are swallowed and reported only as `false`, so verify the store path is writable. Shutdown does not flush; see [setup-and-access](./setup-and-access.md).
- Do not share the store instance: disposing the monitor disposes it.

## Health history

Health sampling (process CPU, drive used space, process memory) starts when the server starts and stops with `server.Stop()`; one sample is kept per minute for 30 days, in memory (and in the `Store`, when set).

`/health/data?period=` accepts `1h` (default for any other value), `1d`, `7d` and `30d`, averaged into 1-minute, 5-minute, 30-minute and 2-hour buckets respectively. It is for trend inspection, not high-resolution telemetry.

## HTTP telemetry (1.7.0-rc3)

The dashboard overview shows request totals (success/client error/server error), latency (avg, p50, p95, p99, max), in-flight requests, per-route stats and recent exceptions. It needs no registration beyond `UseMonitoring`.

- Data is in memory only, kept 7 days; at most 200 distinct routes (the rest roll into `(other routes)`) and the last 20 exceptions.
- Routes are keyed as `METHOD route-path`, or `(unmatched)`. The monitor's own dashboard routes are excluded; connections dropped before a response are not counted.
- The overview endpoint `/data?period=` accepts `1h` (default), `1d`, `7d`.
