# Endpoint documentation practices

For additional details, see the [official Sisk API Documentation extension guide](https://docs.sisk-framework.org/docs/extensions/api-documentation).

Document the contract a consumer can observe. Every attribute here is read from the route method or a request handler's `Execute`/`ExecuteAsync`; there is no XML-comment inference (see [annotations](./annotations.md)).

## Minimum useful public contract

1. Add `[ApiEndpoint]` only to routes intended for the published document.
2. Use `Group` for navigation and `Description` for intent or effects not visible from the route.
3. Add path, query, header, and body attributes only when the contract cannot be inferred accurately.
4. Add `[ApiResponse]` for every status the route can actually return, including validation, authentication, authorization, missing resources, and conflicts. Without any `[ApiResponse]`, OpenAPI invents a default `200`.

## Shared behavior in request handlers

Put attributes common to many routes (an `Authorization` header, a `401`/`403` response) on the handler's `Execute` method instead of repeating them on each route. They merge into every route that uses the handler.

```csharp
public class RequireAuthHandler : IRequestHandler {
    public RequestHandlerExecutionMode ExecutionMode { get; init; } = RequestHandlerExecutionMode.BeforeResponse;

    [ApiHeader("Authorization", IsRequired = true, Description = "Bearer access token.")]
    [ApiResponse(HttpStatusCode.Unauthorized, Description = "Missing or invalid token.")]
    public HttpResponse? Execute(HttpRequest request, HttpContext context) => ...;
}
```

## Prefer types to literal examples

Define request and response types and pass them through `PayloadType`. With a `JsonContentTypeHandler` built from the application's `JsonSerializerOptions`, the example and schema follow the runtime serializer.

```csharp
public sealed record CreateUserRequest(string Name, string Email);
public sealed record UserResponse(long Id, string Name, string Email);

[ApiEndpoint(Group = "Users", Description = "Creates a user when the email is available.")]
[ApiRequest("Creation data.", PayloadType = typeof(CreateUserRequest))]
[ApiResponse(HttpStatusCode.Created, Description = "User created.", PayloadType = typeof(UserResponse))]
[ApiResponse(HttpStatusCode.BadRequest, Description = "Invalid or incomplete body.")]
[ApiResponse(HttpStatusCode.Conflict, Description = "Email already in use.")]
[RoutePost("/users")]
public HttpResponse CreateUser(HttpRequest request) => ...;
```

Use literal examples only for a meaningful variant, an untyped payload, or semantics a schema cannot communicate, and keep them valid for the declared `ExampleLanguage`.

## Accuracy and maintenance

- Do not document speculative statuses.
- State required authentication, headers, pagination, filters, identifier formats, limits, and side effects when relevant.
- Match an `[ApiPathParameter]` name to its route placeholder exactly.
- Revalidate the generated output whenever routes, DTOs, serializer options, or returned statuses change.
