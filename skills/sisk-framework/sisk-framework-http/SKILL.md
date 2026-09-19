---
name: sisk-framework-http
description: Use this skill for Sisk Framework HttpRequest and HttpResponse fundamentals, including request data, headers, bodies, status codes, and response helpers.
---
# Sisk Framework - HTTP Primitives

## Fundamentals — HttpRequest / HttpResponse

### HttpRequest

```csharp
HttpMethod method    = request.Method;
string path          = request.Path;
string fullUrl       = request.FullUrl;
string query         = request.QueryString;
var queryParams      = request.Query;           // StringValueCollection
var routeParams      = request.RouteParameters; // StringValueCollection

// Body
string body          = request.Body;
byte[] rawBody       = request.RawBody;
Stream stream        = request.GetRequestStream(); // single read

bool hasContent      = request.HasContents;
bool loaded          = request.IsContentAvailable;

// Form
var form             = request.GetFormContent();         // NameValueCollection
var multipart        = request.GetMultipartFormContent(); // MultipartObject[]

// Headers
string? auth         = request.Headers.Authorization;

// Context
HttpContext ctx       = request.Context;
var bag              = request.Bag;            // TypedValueDictionary

// Disconnection
CancellationToken dc = request.DisconnectToken;
```

- Configurable size limit in `HttpServerConfiguration.MaximumContentLength` — exceeding returns 413.
- `GetRequestStream()` can only be read once; after that `Body` and `RawBody` become unavailable.
- `HttpContext.Current` / `HttpContext.GetCurrentContext()` gets the current thread's context.
- `request.Bag.Set<T>()` / `request.Bag.Get<T>()` to pass typed data between handlers.

### HttpResponse

```csharp
// Basic construction
var res = new HttpResponse(200) {
    Content = new StringContent(json, Encoding.UTF8, "application/json")
};

// Fluent API
return new HttpResponse()
    .WithStatus(HttpStatusCode.Created)
    .WithHeader("Location", $"/users/{id}")
    .WithContent(JsonContent.Create(user))
    .WithCookie("session", token, expiresAt: DateTime.UtcNow.AddDays(7));

// Redirect
return new HttpResponse(301).WithHeader("Location", "/login");

// Chunked
var res = new HttpResponse { SendChunked = true };

// Response stream (large files)
var responseStream = request.GetResponseStream();
responseStream.SendChunked = true;
responseStream.SetStatus(200);
responseStream.SetHeader(HttpKnownHeaderNames.ContentType, "application/zip");
using var fileStream = File.OpenRead("file.zip");
fileStream.CopyTo(responseStream.ResponseStream);
return responseStream.Close();
```

**Implicit return types** — configure the router to convert custom objects:
```csharp
router.RegisterValueHandler<MyResult>(result =>
    new HttpResponse { Content = JsonContent.Create(result) });

// fallback for any type (must be registered last)
router.RegisterValueHandler<object>(obj =>
    new HttpResponse { Content = JsonContent.Create(obj) });
```

- `HttpHeaderCollection.Add()` adds without replacing; `.Set()` / indexer replaces.
- `response.GetHeaderValue("Content-Type")` searches in `Headers` and `Content.Headers`.

---
