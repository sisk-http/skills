# Runtime and errors

For additional details, see the [official Sisk JSON-RPC documentation](https://docs.sisk-framework.org/docs/extensions/json-rpc).

## Request handling

Each transport reads one request object, finds the method, binds `params`, invokes the method by reflection, and replies with a `JsonRpcResponse` carrying the request `id`.

The request check is strict: `jsonrpc` must be `"2.0"`, `method` must be non-empty, `id` must be a string, number, null or absent, and `params` is required and must be an array or an object. Batch arrays are not supported; over POST an array body fails as `-32603`.

`Task<T>` results are awaited synchronously with `GetResult()`, so async methods still hold a request thread. `IAsyncEnumerable<T>` results are enumerated to completion and serialized as one array. Return `Task<T>` rather than plain `Task`: registration accepts it, but the dispatcher reinterprets every task as `Task<object>` to read its result.

## Notifications and transports

- POST: a request without `id`, or with `id: null`, still runs the method. Success returns `202 Accepted` with no body; failures return `200` with an error object.
- GET: always answers `200` and requires `id` in the query string.
- WebSocket: every message gets a response, including notifications. A message that is not valid JSON gets `-32600` with a null `id`. A valid JSON message with an invalid request shape, such as missing `params`, is not converted to an error response; the exception escapes the receive loop.

## Error codes

| Code | Produced when |
|---|---|
| `-32600` | POST `Content-Type` is present but is not `application/json`; a WebSocket message is not valid JSON. |
| `-32601` | No registered method has the requested name. |
| `-32602` | The argument count is outside the required-to-total range (unknown extra properties in a named object count), or a named argument is null or missing for a non-optional parameter that is not `Nullable<T>`. |
| `-32603` | Any other exception: malformed POST or GET input, invalid request shape, parameter conversion failure, or an exception from the method. The message is `Exception.Message` (the inner exception's for method failures), `data` is null, and `id` is `"0"` when the request could not be parsed. |

`-32700` is never produced; malformed JSON surfaces as `-32603` over POST and GET and as `-32600` over WebSocket.

## Custom errors

`JsonRpcException(message)` uses code `1`; `JsonRpcException(message, code, data)` sets the code and a `data` object serialized into `error.data`. Choose codes deliberately; `-32000` to `-32099` is the JSON-RPC range for server-defined errors.

In 1.6.3 the dispatcher catches `JsonRpcException` only when it comes out of a task. An exception thrown synchronously reaches it wrapped in `TargetInvocationException`, so the client gets `-32603` with the message only, losing `code` and `data`. Make methods that raise custom codes `async Task<T>`:

```csharp
[WebMethod]
public async Task<Order> GetOrder(string id)
{
    return await orders.FindAsync(id)
        ?? throw new JsonRpcException("Order not found.", code: -32010, data: new { Id = id });
}
```

Unexpected exceptions thrown by a method are written to `ErrorsLogsStream` and their message is returned to the client. With `HttpServerConfiguration.ThrowExceptions` set to `true` they are rethrown to Sisk instead of producing a JSON-RPC error. Keep internal exceptions, stack traces, credentials and storage details out of messages and `data`; to hide an internal failure, catch it inside the method and throw a `JsonRpcException` with a generic message.

## Concurrency and limits

Method target instances are shared across concurrent requests. Keep mutable state thread-safe, avoid request state on the service object, and use `HttpContext.Current` or request-scoped dependencies only during the active invocation. No cancellation token is injected into method signatures; pass cancellation through application abstractions when a method performs I/O.

`MaximumContentLength` is server-wide and defaults to 100 MB in core 1.7.0. A POST declaring a larger `Content-Length` receives `413` before reaching the extension, a chunked body that exceeds it fails as `-32603`, and core also uses it as the maximum WebSocket message size. Lower it for RPC-only servers, or set `0` for unlimited.
