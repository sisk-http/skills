---
name: sisk-framework
description: Use when a Sisk Framework task spans routing, HTTP, middleware, or realtime APIs and needs guidance on which focused skill to read.
---
# Sisk Framework — Core

Sisk is an HTTP framework for .NET that uses `HttpServer`, `Router`, `Route`, `HttpRequest`, and `HttpResponse` as fundamental blocks. Always prefer the fluent API (`.With*`) and C#'s functional patterns when working with Sisk.

Full reference: https://docs.sisk-framework.org/docs/summary.g

---

## Select the relevant skill

Read only the skill needed for the task; go directly to it when the topic is already known. These examples use Sisk Framework types, not the low-level Cadente host API. Confirm signatures against the project's installed version before changing code.

- [sisk-framework-routing](./sisk-framework-routing/SKILL.md): route mapping, route parameters, modules, attributes, prefixes, constraints, and request metadata
- [sisk-framework-http](./sisk-framework-http/SKILL.md): HttpRequest and HttpResponse fundamentals
- [sisk-framework-handlers](./sisk-framework-handlers/SKILL.md): request handlers, middleware flow, compression, and server handlers
- [sisk-framework-realtime](./sisk-framework-realtime/SKILL.md): WebSockets, SSE, and LogStream

For other Sisk packages, use [Cadente](../sisk-cadente/SKILL.md) for the HTTP engine, [MCP](../sisk-model-context-protocol/SKILL.md) for tool servers, [Documenting](../sisk-document/SKILL.md) for endpoint annotations, or [Monitoring](../sisk-monitoring/SKILL.md) for the embedded dashboard.

Validate an implementation with the affected project's build and a focused check of the changed route or lifecycle behavior. A review request remains read-only; loading a skill does not authorize a refactor.
