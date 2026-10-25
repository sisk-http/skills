---
name: sisk-json-rpc
description: Use when exposing JSON-RPC 2.0 methods with Sisk.JsonRpc, mapping its HTTP or WebSocket transports, configuring the serializer or documentation, or diagnosing protocol errors.
---
# Sisk.JsonRpc

`Sisk.JsonRpc` (namespace `Sisk.JsonRPC`) is an extension for Sisk.HttpServer, documented as experimental. It registers `[WebMethod]` methods or delegates in a `JsonRpcHandler` and exposes them as `RouteAction` transports that you map on ordinary Sisk routes. Mapped routes carry no authentication or rate limiting of their own; add them, and tune body-size limits, as for any other endpoint.

- [Setup](./references/setup.md): read when installing the package, registering methods, mapping `HttpPost`, `HttpGet` or `WebSocket`, binding parameters, configuring the serializer, or generating method documentation.
- [Runtime and errors](./references/runtime-and-errors.md): read when throwing or debugging JSON-RPC errors, handling notifications, comparing transport behavior, or reviewing concurrency and payload limits.

For additional details, see the [official Sisk JSON-RPC documentation](https://docs.sisk-framework.org/docs/extensions/json-rpc).
