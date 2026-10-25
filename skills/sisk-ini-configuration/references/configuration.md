# Configuration

For additional details, see the [official Sisk INI Configuration documentation](https://docs.sisk-framework.org/docs/extensions/ini-configuration).

## INI syntax

The reader never throws on syntax mistakes; it ignores what it does not understand.

- Section and key names are case-insensitive (`IniReader.IniNamingComparer`, a static `StringComparer` defaulting to `InvariantCultureIgnoreCase`). Names and unquoted values are trimmed.
- Properties before the first section belong to `document.Global`. Sections with the same name are merged.
- A line starting with `#` or `;` is a comment. An unquoted value is cut at its first `#` or `;`, so quote values that contain them, such as connection strings.
- Values wrapped in `'` or `"` keep their line breaks and any comment characters; quotes cannot be escaped, and a comment may follow the closing quote.
- `Key = <<<EOF` starts a heredoc: following lines are taken literally until a line containing only `EOF`, without the final line break. `<<<` without a delimiter is a literal value.
- A key without `=` gets an empty value.
- A repeated key keeps every value: `GetMany` returns all of them, `GetOne` returns the last one.

```csharp
IniDocument document = IniDocument.FromString(text);
string? name = document.Global.GetOne("name");
string[] colors = document.GetSection("display")?.GetMany("color") ?? [];
```

- Load with `IniDocument.FromString`, `FromFile(path, encoding, throwIfNotExists = true)` (use `throwIfNotExists: false` to get an empty document), or `FromStream`.
- Use `IniReader` and `IniWriter` when the application owns the `TextReader` or `TextWriter`.
- Write with `document.ToString()`, `SaveTo(Stream)`, or `SaveTo(path)`. `SaveTo(path)` opens the file with `File.OpenWrite` without truncating it, so overwriting a longer file leaves stale trailing content; prefer `File.WriteAllText(path, document.ToString())`.

## Portable host sections

`IniConfigurationReader` reads only the keys below; any other key or section is silently ignored, so a misspelled key has no effect.

**`[Server]`**

- `Listen`: repeatable `ListeningPort` URI. Entries are appended to the single listening host; with none, Sisk picks a random port when building.
- `MaximumContentLength`: bytes, parsed as `long`. When omitted the reader leaves the server default (100 MB since 1.7), unlike the JSON portable reader, which assigns `0`. Use `0` for unlimited.
- `IncludeRequestIdHeader`, `ThrowExceptions`: a value equal to `true` (case-insensitive) enables them; any other value disables them.
- `AccessLogsStream`, `ErrorsLogsStream`: `console` or a log file path.

There is no `Encoding` key and no way to configure more than one listening host, its label, or SSL from INI; do that in code.

**`[Cors]`** (the section name is case-insensitive)

- `AllowMethods`, `AllowHeaders`, `ExposeHeaders`: comma-separated lists.
- `AllowOrigin`: a single origin. `AllowOrigins`: repeatable, each entry comma-separated.
- `AllowCredentials`: `true` enables it. Do not combine it with a wildcard origin.
- `MaxAge`: integer seconds.

**`[Parameters]`**: every key becomes a string in `host.Parameters`; for a repeated key the last value wins.

```ini
[Server]
Listen = http://localhost:8080/
Listen = http://localhost:8081/
MaximumContentLength = 8388608
ThrowExceptions = false
AccessLogsStream = console

[Cors]
AllowMethods = GET, POST
AllowHeaders = Content-Type, Authorization
AllowOrigin = https://example.test
MaxAge = 3600

[Parameters]
ServiceName = catalog
Database = "server=localhost;user=app;"
```

The portable builder reads the file once while building the host. It is not a live reload service; reloading requires an explicit policy because edits do not mutate an already-built host.
