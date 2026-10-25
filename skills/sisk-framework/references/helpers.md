# Helpers

For additional details, see the [official Sisk.Core.Helpers API documentation](https://docs.sisk-framework.org/api/Sisk.Core.Helpers.html).

Use helpers from `Sisk.Core.Helpers` instead of repeating size, URL-path, filesystem-path, or MIME logic.

## `SizeHelper`

The binary unit constants make byte limits readable:

```csharp
configuration.MaximumContentLength = 8 * SizeHelper.UnitMb;
```

`UnitKb`, `UnitMb`, `UnitGb`, `UnitTb`, `UnitPb`, and `UnitEb` are byte counts. `Parse("8 MB")` converts a human-readable size to bytes, and `HumanReadableSize(byteCount)` formats a byte count for display or logs.

## `PathHelper`

Use `CombinePaths` for URL paths and `FilesystemCombinePaths` for filesystem paths; they have different normalization rules.

```csharp
string route = PathHelper.CombinePaths("/api", "orders", orderId.ToString());
string file = PathHelper.FilesystemCombinePaths(dataDirectory, "orders", "index.json");
```

`NormalizePath`, `Split`, and `Pop` operate on path segments. `EnsureFileDirectoryExistance` and `EnsureDirectoryExistance` create missing directories and return whether the directory already existed. These methods do not authorize user-controlled paths; enforce the application's root and traversal policy separately.

## `MimeHelper`

Resolve MIME types from an extension or a path and reverse-map known MIME types when needed:

```csharp
string contentType = MimeHelper.GetMimeType(Path.GetExtension(fileName));
string? extension = MimeHelper.GetFileExtension(contentType);
```

`GetMimeType` treats a value starting with `.` as an extension and a value containing a directory separator as a path. A bare file name such as `report.pdf` is not recognized and returns the default MIME type.

`IsPlainTextFile`, `IsPlainTextMimeType`, and `IsBrowserKnownInlineMimeType` classify known values. Treat MIME lookup as metadata, not proof that untrusted bytes match the claimed format.
