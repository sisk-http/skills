# Setup

For additional details, see the [official Sisk INI Configuration documentation](https://docs.sisk-framework.org/docs/extensions/ini-configuration) and [Service Providers](https://docs.sisk-framework.org/docs/extensions/service-providers).

## Packages

- `Sisk.IniConfiguration`: `IniConfigurationReader` plus the INI document API (it references the Core package).
- `Sisk.IniConfiguration.Core`: only `IniDocument`, `IniReader`, and `IniWriter`, for use outside a Sisk host.

```bash
dotnet add package Sisk.IniConfiguration
```

## Registering the reader

```csharp
using var host = HttpServer.CreateBuilder()
    .UsePortableConfiguration(configuration =>
    {
        configuration.WithConfigFile("server.ini", createIfDontExists: false);
        configuration.WithConfigReader<IniConfigurationReader>();
    })
    .UseRouter(router => router.MapGet("/", request => new HttpResponse("ok")))
    .Build();
```

- Call `UsePortableConfiguration` first in the builder chain: it overrides most settings configured before it.
- Always call `WithConfigFile`. Without it the builder looks for `service-config.json`, the default name of the JSON reader.
- `WithConfigFile(filename, createIfDontExists = false, lookupDirectories = CurrentDirectory)` searches the process current directory by default. Add `ConfigurationFileLookupDirectory.AppDirectory` (or `All`) to also search the executable directory.
- A missing file throws `ArgumentException`. With `createIfDontExists: true` Sisk creates an empty file and continues with defaults, so avoid it for required production configuration.
- `WithConfigReader` accepts an `IConfigurationReader` instance or a type with a parameterless constructor.

## Failures at startup

`IniConfigurationReader` wraps parse failures (for example a non-numeric `Server.MaximumContentLength`) in an exception that names the failing key. `UsePortableConfiguration` catches configuration exceptions, prints them, and calls `Environment.Exit(2)`. Set the static `HttpServerHostContextBuilder.CatchConfigurationExceptions` to `HttpServerHostContextBuilderExceptionMode.Throw` when the exception must propagate, such as in tests.

## Reading parameters

Application values from the `[Parameters]` section are available in `HttpServerHostContext.Parameters` and become read-only once the portable configuration has been read. Validate required ones once at startup with `EnsureNotNull`, `EnsureNotNullOrEmpty`, or `GetValueOrThrow`, then expose a typed application configuration object instead of parsing raw strings in routes.

## Related packages

`Sisk.ServiceProvider` was removed from the supported packages in 1.7.0; use `UsePortableConfiguration` from `Sisk.HttpServer` instead.
