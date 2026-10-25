# Deferred actions

For additional details, see the official Sisk documentation on [deferred actions](https://docs.sisk-framework.org/docs/fundamentals/responses#deferred-actions).

`HttpContext.EnqueueDeferredAction` schedules work after the response is sent but before the request context and its bag values are disposed. Enqueue the action before returning the response:

```csharp
router.MapPost("/orders", async request =>
{
    Order order = await CreateOrderAsync(request.DisconnectToken);
    AuditWriter audit = request.Context.RequestBag.GetOrAdd(CreateAuditWriter);

    request.Context.EnqueueDeferredAction(
        cancellation => audit.WriteOrderAcceptedAsync(order.Id, cancellation),
        timeout: TimeSpan.FromSeconds(5));

    return new HttpResponse(202);
});
```

The route returns immediately to the router; Sisk sends that response, executes the deferred action, and only then disposes the context and disposable `RequestBag` values. Request-scoped dependencies such as `audit` are therefore valid inside the queued action, but not after that action completes.

Queued actions run sequentially in enqueue order. When a request has deferred actions, Sisk disables keep-alive for that response. The connection loop waits for the queue, so deferred work is not fire-and-forget.

Available overloads accept synchronous actions, `Func<Task>`, an external cancellation token, or a timeout-backed cancellation token. Deferred code must observe cancellation and remain bounded. The timeout overload has `timeout = default`, which is `TimeSpan.Zero`: the token is already cancelled and the action is skipped, so always pass a positive timeout.

The same ambient `HttpContext` and `RequestBag` remain available while an action executes. They become invalid after the deferred queue finishes. Prefer capturing immutable identifiers and payloads instead of the whole request. Never store or use request-scoped resources after completion.

The client has already received the response, so a deferred failure cannot change it. The queue runs inside the request's exception handling: an exception is treated like any other unhandled server error (rethrown when `ThrowExceptions` is `true`) and the actions still queued after it are skipped. Catch and log inside each action when later actions must run.

Use deferred actions for short, non-critical post-response work whose request-scoped resources and failure model are acceptable. Use a durable queue for work that must survive process termination, retry independently, scale separately, or run for an unbounded time.
