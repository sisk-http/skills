---
name: sisk-cadente
description: Use when enabling, configuring, or debugging Sisk.Cadente, the managed HTTP/1.1 listener, either as a Sisk engine (CadenteHttpServerEngine) or as a standalone HttpHost with handlers, TLS/mTLS, and timeouts.
---
# Sisk.Cadente

> **Experimental (Beta):** the official documentation does not recommend Cadente for critical production environments, and its API and behavior may change. It implements HTTP/1.1 (no HTTP/2 or QUIC), and `Sisk.Cadente` / `Sisk.Cadente.CoreEngine` target .NET 9 and .NET 10 only, so they cannot be referenced from a .NET 8 project.

For a Sisk application, use `CadenteHttpServerEngine` from `Sisk.Cadente.CoreEngine` as the primary integration path. Use standalone `HttpHost` (package `Sisk.Cadente`, no Sisk dependency) only when direct, low-level control of Cadente connections is actually required.

Read the matching reference before changing code:

- [Setup and integration](references/setup.md): read when adding the package, wiring the engine into a Sisk server, configuring TLS/mTLS or engine-level timeouts, or starting a standalone host.
- [Host APIs](references/host-apis.md): read when writing or debugging a standalone `HttpHostHandler`: request bodies, response framing, keep-alive, and request validation.
- [Reference types](references/reference-types.md): read when using `HttpHostClient`, `HttpHeaderList`/`HttpHeader`, `HttpsOptions`, or `HttpHostTimeoutManager`, including their defaults.

For additional details, see the [official Sisk Cadente documentation](https://docs.sisk-framework.org/docs/cadente).
