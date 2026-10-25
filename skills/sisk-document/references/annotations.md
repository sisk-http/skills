# Endpoint annotations

For additional details, see the [official Sisk API Documentation extension guide](https://docs.sisk-framework.org/docs/extensions/api-documentation).

Import `Sisk.Documenting.Annotations`. All attributes target methods. Only `[ApiEndpoint]` is single-use; every other attribute is repeatable. A route is included only when its method has `[ApiEndpoint]`; other documentation attributes on an unmarked route are ignored.

The reader also collects every attribute except `[ApiEndpoint]` from the `Execute` (or `ExecuteAsync` for `AsyncRequestHandler`) method of each request handler attached to the route. Use this to document what a handler adds to all its routes, such as an authentication header or a `401` response.

## Operation identity

`[ApiEndpoint]` has an empty constructor and a `(string name)` constructor. `Name` falls back to the route name, then the method name. In OpenAPI, `Name` becomes the operation `summary` and `Group` becomes its tag; in HTML, endpoints without `Group` are listed under "Requests". Endpoints are sorted by `Order` (default `0`), then by `Path` descending.

`InheritDescriptionFromXmlDocumentation` exists on the attribute (default `true`) but nothing in 1.7.0-rc3 reads it. Set `Description` explicitly instead of relying on XML comments.

```csharp
[ApiEndpoint(Group = "Users", Description = "Returns the authenticated user's profile.")]
[RouteGet("/users/me")]
public HttpResponse GetCurrentUser(HttpRequest request) => ...;
```

`Description` is rendered as Markdown (CommonMark) in the HTML exporter.

## Request and responses

`[ApiRequest(string description)]` documents the incoming body. `[ApiResponse(HttpStatusCode statusCode)]` documents one possible response; add one per status the route can return. Both support `Description`, `PayloadType`, `Example`, `ExampleLanguage`, and `JsonSchema`.

```csharp
[ApiEndpoint("Create user", Group = "Users")]
[ApiRequest("New user data.", PayloadType = typeof(CreateUserRequest))]
[ApiResponse(HttpStatusCode.Created, Description = "User created.", PayloadType = typeof(UserResponse))]
[ApiResponse(HttpStatusCode.Conflict, Description = "Email already registered.")]
[RoutePost("/users")]
public HttpResponse CreateUser(HttpRequest request) => ...;
```

When `PayloadType` is set, the example comes from `ApiGenerationContext.BodyExampleTypeHandler` and overrides a literal `Example`; the schema comes from `ContentSchemaTypeHandler` unless `JsonSchema` is set. Without a configured handler, `PayloadType` produces nothing.

## Parameters and headers

- `[ApiPathParameter("id", Type = "integer", Description = "...")]` documents a route `<id>` placeholder; always `required` in OpenAPI.
- `[ApiQueryParameter("page", Type = "integer", IsRequired = false, Description = "...")]` documents query input.
- `[ApiHeader("Authorization", IsRequired = true, Description = "...")]` documents a header; OpenAPI always types it as `string`.
- `[ApiParameter("name", "string", IsRequired = true, Description = "...")]` documents one body field when no payload type fits.
- `[ApiParametersFrom(typeof(SearchRequest))]` expands a type into body fields through `ParameterExampleTypeHandler`; without that handler it adds none.

`Type` and `TypeName` are free text. `OpenApiExporter` maps `int`/`int32`/`integer`/`long`/`int64` to `integer`, `float`/`single`/`double`/`decimal` to `number`, `bool`/`boolean` to `boolean`, and anything else (including unknown names) to `string`.

## Contextual parameter examples

Use `[ApiParameterExample(parameterName, title)]` for alternate values of a body field already created by `[ApiParameter]` or `[ApiParametersFrom]`.

```csharp
[ApiEndpoint("Search products", Group = "Catalog")]
[ApiParameter("query", "string", IsRequired = true)]
[ApiParameterExample("query", "Text search", ExampleLanguage = "text", Example = "wireless keyboard")]
[ApiParameterExample("query", "SKU search", ExampleLanguage = "text", Example = "sku:KB-1042")]
[RoutePost("/products/search")]
public HttpResponse SearchProducts(HttpRequest request) => ...;
```

`Description`, `ExampleLanguage`, and `Example` are optional. Matching is ordinal and case-sensitive. `OpenApiExporter` uses the first non-null example as the property `example` and keeps all scenarios in `x-examples`. HTML and `LlmsApiExporter` group scenarios under the matching body field. An example that matches no body field is kept in the model but not rendered.

Do not use it for a whole body, path, query, or header parameter; use `[ApiRequest]` for a full payload.

## Filtering and rewriting annotations

Set `ApiGenerationContext.Handler` to a subclass of `ApiDocumentationHandler` to filter or adjust the collected model per `Route`. Override any `HandleApiEndpoint{Response|Parameter|ParameterExample|Header|PathParameter|RequestExample|QueryParameter}(value, route)` and return `false` to drop that item, or mutate `value` and return `true`. Override `ShouldCreateApiEndpoint(endpoint, route)` and return `false` to exclude a whole endpoint.
