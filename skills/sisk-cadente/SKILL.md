---
name: sisk-cadente
description: Use when choosing between Sisk.Cadente engine integration and standalone HTTP/1.1 hosting, or locating its host, TLS, and timeout guidance.
---
# Sisk.Cadente

Cadente is an **experimental** fully managed HTTP/1.1 server for Sisk, replacing the default `System.Net.HttpListener`. Key advantages: native SSL/TLS on all platforms, consistent cross-platform behavior, no dependency on `HttpListener`.

> **Status: Beta** — API may change. Not recommended for critical production without testing.

Reference: https://docs.sisk-framework.org/docs/cadente  
Packages: `Sisk.Cadente` (standalone), `Sisk.Cadente.CoreEngine` (Sisk integration)

---

## Select the relevant skill

Read only the matching skill; go directly to it when the topic is known. Cadente host types differ from [Sisk Framework HTTP types](../sisk-framework/sisk-framework-http/SKILL.md). Check APIs and defaults against the installed version; this experimental package may change.

- [sisk-cadente-setup](./sisk-cadente-setup/SKILL.md): Cadente as Sisk engine and standalone server setup
- [sisk-cadente-host](./sisk-cadente-host/SKILL.md): HttpHost, HttpHostHandler, HttpHostContext, request, and response APIs
- [sisk-cadente-reference](./sisk-cadente-reference/SKILL.md): client, headers, HTTPS options, timeouts, and stream lifetime rules

Validate changed behavior with a build and a focused local request covering the affected framing, TLS, timeout, or lifecycle path. Do not start listeners or change certificates merely to review this guidance.
