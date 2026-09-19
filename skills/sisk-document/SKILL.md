---
name: sisk-document
description: Use para documentar endpoints Sisk com Sisk.Documenting.Annotations, incluindo ApiEndpoint, ApiRequest, ApiResponse, headers e parâmetros.
---
# Instructions for API Documentation

Use `Sisk.Documenting.Annotations` when documenting Sisk endpoints. Inspect the selected routes, request/response types, and handlers first; describe the implemented contract rather than inventing fields, status codes, or authentication requirements. A documentation request does not authorize changing runtime behavior or exposing internal routes.

## Available Attributes

### `[ApiEndpoint]`
Defines the name, description, and group of an endpoint.

```csharp
[ApiEndpoint("Create User", Description = "Creates a new user in the system", Group = "Users")]
[RoutePost("/users")]
public HttpResponse CreateUser(HttpRequest request) { ... }
```

**Properties:**
- `Name` (required): Human-readable name of the endpoint
- `Description`: Description of what the endpoint does
- `Group`: Logical grouping of the endpoint (e.g., "Users", "Products")
- `InheritDescriptionFromXmlDocumentation`: If `true`, inherits description from XML doc (default: `true`)

---

### `[ApiRequest]`
Documents the expected request body.

```csharp
[ApiRequest("User data to be created", PayloadType = typeof(CreateUserRequest))]
[RoutePost("/users")]
public HttpResponse CreateUser(HttpRequest request) { ... }
```

**Properties:**
- `Description` (constructor arg): Description of the request body
- `PayloadType`: Type used to generate the example and JSON schema automatically
- `Example`: Literal example body (string)
- `ExampleLanguage`: Language of the example (e.g., "json", "xml")
- `JsonSchema`: Raw JSON schema string describing the expected payload

---

### `[ApiResponse]`
Documents the possible responses of the endpoint.

```csharp
[ApiResponse(HttpStatusCode.Created, Description = "User created successfully", PayloadType = typeof(UserResponse))]
[ApiResponse(HttpStatusCode.BadRequest, Description = "Invalid data")]
[ApiResponse(HttpStatusCode.Conflict, Description = "User already exists")]
[RoutePost("/users")]
public HttpResponse CreateUser(HttpRequest request) { ... }
```

**Properties:**
- `StatusCode` (constructor arg): HTTP status code of the response
- `Description`: Description of the response
- `PayloadType`: Type used to generate the example and JSON schema automatically
- `Example`: Literal example response (string)
- `ExampleLanguage`: Language of the example
- `JsonSchema`: Raw JSON schema string describing the response payload

---

### `[ApiHeader]`
Documents expected request headers.

```csharp
[ApiHeader("Authorization", Description = "JWT authentication token", IsRequired = true)]
[ApiHeader("X-Request-Id", Description = "Unique request ID for tracking")]
[RouteGet("/users/me")]
public HttpResponse GetCurrentUser(HttpRequest request) { ... }
```

**Properties:**
- `HeaderName` (required): Name of the header
- `Description`: Description of the header
- `IsRequired`: Whether the header is required (default: `false`)

---

### `[ApiPathParameter]`
Documents path parameters.

```csharp
[ApiPathParameter("userId", Type = "integer", Description = "Unique user ID")]
[RouteGet("/users/<userId>")]
public HttpResponse GetUser(HttpRequest request) { ... }
```

**Properties:**
- `Name` (required): Name of the parameter in the route
- `Type`: Type of the parameter (string, integer, uuid, etc.)
- `Description`: Description of the parameter

---

### `[ApiQueryParameter]`
Documents query string parameters.

```csharp
[ApiQueryParameter("page", Type = "integer", Description = "Page number")]
[ApiQueryParameter("limit", Type = "integer", Description = "Number of items per page")]
[ApiQueryParameter("search", Description = "Search term")]
[RouteGet("/users")]
public HttpResponse ListUsers(HttpRequest request) { ... }
```

**Properties:**
- `Name` (required): Name of the parameter
- `Type`: Type of the parameter
- `Description`: Description of the parameter
- `IsRequired`: Whether the parameter is required (default: `false`)

---

### `[ApiParameter]`
Documents generic parameters.

```csharp
[ApiParameter("email", "string", Description = "User email", IsRequired = true)]
[RoutePost("/users")]
public HttpResponse CreateUser(HttpRequest request) { ... }
```

**Properties:**
- `Name` (required): Name of the parameter
- `TypeName` (required): Type of the parameter
- `Description`: Description of the parameter
- `IsRequired`: Whether it is required (default: `false`)

---

### `[ApiParametersFrom]`
Generates parameters automatically from a type.

```csharp
[ApiParametersFrom(typeof(CreateUserRequest))]
[RoutePost("/users")]
public HttpResponse CreateUser(HttpRequest request) { ... }
```

---

## Wiring documentation generation

Documentation is exposed via the `UseApiDocumentation` extension on `HttpServerHostContextBuilder` (from `Sisk.Documenting`). Default route is `/api/docs`.

```csharp
using Sisk.Documenting;
using Sisk.Documenting.Html;       // HtmlDocumentationExporter — separate package: Sisk.Documenting.Html

HttpServer.CreateBuilder()
    .UseApiDocumentation(
        new ApiGenerationContext {
            ApplicationName = "MyApp",
            ApplicationVersion = "1.0.0",
            ApplicationDescription = "..."
        },
        "/docs",                              // route path (optional; defaults to /api/docs)
        new HtmlDocumentationExporter {       // optional; defaults to OpenApiExporter
            PageTitle = "MyApp — API"
        })
    .UseRouter(...)
    .Build();
```

**Available exporters:**
- `OpenApiExporter` (built-in in `Sisk.Documenting`): emits OpenAPI/Swagger JSON.
- `HtmlDocumentationExporter` (in `Sisk.Documenting.Html` — separate NuGet package): emits a self-contained HTML page with `PageTitle`, `Style`, `Script`, `Header`, `Footer` customization points.

The `ApiGenerationContext` also exposes `IExampleBodyTypeHandler`, `IExampleParameterTypeHandler`, and `IContentSchemaTypeHandler` slots if you need custom example/schema generation.

---

## Best Practices

### 1. Always document public endpoints
Every publicly exposed endpoint within the requested scope should have at least:
- `[ApiEndpoint]` with name and description
- `[ApiResponse]` for the main status codes

### 2. Create records for requests and responses
Prefer specific records when designing structured requests and responses. Reuse existing types for documentation-only work; do not refactor the endpoint solely to introduce a DTO. If no type exists, use a literal example as shown below or propose records as a separate change:

```csharp
// Request records
public record CreateUserRequest(
    string Name,
    string Email,
    string Password
);

// Response records
public record UserResponse(
    int Id,
    string Name,
    string Email,
    DateTime CreatedAt
);

// Usage in the endpoint
[ApiEndpoint("Create User", Group = "Users")]
[ApiRequest("New user data", PayloadType = typeof(CreateUserRequest))]
[ApiResponse(HttpStatusCode.Created, PayloadType = typeof(UserResponse))]
[ApiResponse(HttpStatusCode.BadRequest, Description = "Invalid input data")]
[RoutePost("/users")]
public HttpResponse CreateUser(HttpRequest request) { ... }
```

### 3. Provide practical examples when there are no defined types
When no suitable type exists, provide literal examples that match the actual payload:

```csharp
[ApiRequest("User data",
    ExampleLanguage = "json",
    Example = """
    {
        "name": "John Doe",
        "email": "john@example.com",
        "password": "password123"
    }
    """)]
[ApiResponse(HttpStatusCode.OK,
    ExampleLanguage = "json",
    Example = """
    {
        "id": 1,
        "name": "John Doe",
        "email": "john@example.com",
        "createdAt": "2025-01-15T10:30:00Z"
    }
    """)]
```

### 4. Document all possible status codes
List responses supported by the route and its applicable handlers, including documented shared error behavior. The following is an illustrative set, not a checklist to copy onto every endpoint:

```csharp
[ApiResponse(HttpStatusCode.OK, Description = "Success")]
[ApiResponse(HttpStatusCode.BadRequest, Description = "Invalid parameters")]
[ApiResponse(HttpStatusCode.Unauthorized, Description = "Not authenticated")]
[ApiResponse(HttpStatusCode.Forbidden, Description = "No permission")]
[ApiResponse(HttpStatusCode.NotFound, Description = "Resource not found")]
[ApiResponse(HttpStatusCode.InternalServerError, Description = "Internal error")]
```

### 5. Logically group endpoints
Use the `Group` property to organize related endpoints:

```csharp
[ApiEndpoint("List Users", Group = "Users")]
[ApiEndpoint("List Products", Group = "Products")]
[ApiEndpoint("Create Order", Group = "Orders")]
```

### 6. Document authentication headers
Always indicate when an endpoint requires authentication. Check whether its handler already supplies inherited header annotations before duplicating them on each route:

```csharp
[ApiHeader("Authorization", Description = "Bearer {token}", IsRequired = true)]
```

### 7. Add XML Documentation to Records
Always include XML documentation for records used in requests and responses, especially if they are missing or incomplete. This helps in generating comprehensive API documentation.

```csharp
/// <summary>
/// Represents the data required to create a new user.
/// </summary>
/// <param name="Name">The full name of the user.</param>
/// <param name="Email">The email address of the user.</param>
/// <param name="Password">The password for the user account.</param>
public record CreateUserRequest(
    string Name,
    string Email,
    string Password
);
```

---

## Completion

Compare annotations and examples with the selected route's body, path/query parameters, headers, and responses, including inherited handler metadata. Build the affected project; when an existing documentation exporter can run locally without external side effects, inspect its output for missing or duplicate fields. Report any generator validation not performed. Do not add unrelated routes or DTO refactors to complete documentation.
