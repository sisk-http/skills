---
name: sisk-ini-configuration
description: Use when loading Sisk host settings from an INI file with Sisk.IniConfiguration and UsePortableConfiguration, or when parsing and writing INI documents with Sisk.IniConfiguration.Core.
---
# Sisk.IniConfiguration

`Sisk.IniConfiguration` provides `IniConfigurationReader`, which plugs the INI reader into Sisk's portable host configuration. `Sisk.IniConfiguration.Core` is the standalone INI document reader and writer, without the Sisk dependency.

- [Setup](./references/setup.md): read when registering the reader with `UsePortableConfiguration`, choosing the package, locating the file, or handling configuration failures at startup.
- [Configuration](./references/configuration.md): read when writing or parsing INI text, or when you need the exact `Server`, `Cors`, and `Parameters` keys the reader applies.

For additional details, see the [official Sisk INI Configuration documentation](https://docs.sisk-framework.org/docs/extensions/ini-configuration).
