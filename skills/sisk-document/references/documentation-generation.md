# Generation and validation

For additional details, see the [official Sisk API Documentation extension guide](https://docs.sisk-framework.org/docs/extensions/api-documentation).

`UseApiDocumentation(context, routerPath = "/api/docs", exporter = null)` extends `HttpServerHostContextBuilder` (namespace `Sisk.Documenting`). It registers a server handler that, when the router is bound to the server, reads the routes once and maps a `GET` route at `routerPath`. The default exporter is `OpenApiExporter`. The parameterless overload uses an empty `ApiGenerationContext`. Call it once per exposed format, each with its own path; annotation or context changes need a restart.

## Serialization context

Prefer payload types and one `JsonContentTypeHandler` that uses the same `JsonSerializerOptions` as the application.

```csharp
using Sisk.Documenting.Content;
using Sisk.Documenting.Exporters;
using Sisk.Documenting.Html;

var jsonOptions = new JsonSerializerOptions {
    TypeInfoResolver = AppJsonSerializerContext.Default
};
var contentHandler = new JsonContentTypeHandler(AppJsonSerializerContext.Default, jsonOptions);

var documentation = new ApiGenerationContext {
    ApplicationName = "My API",
    ApplicationVersion = "1.0.0",
    ApplicationDescription = "Public API.",
    BodyExampleTypeHandler = contentHandler,
    ParameterExampleTypeHandler = contentHandler,
    ContentSchemaTypeHandler = contentHandler
};

builder
    .UseApiDocumentation(documentation, "/openapi.json", new OpenApiExporter())
    .UseApiDocumentation(documentation, "/docs", new HtmlDocumentationExporter { PageTitle = "My API" });
```

- `IExampleBodyTypeHandler`, `IExampleParameterTypeHandler`, and `IContentSchemaTypeHandler` are independent slots on `ApiGenerationContext`; all default to `null`, and then `PayloadType` and `[ApiParametersFrom]` produce nothing. `JsonContentTypeHandler` implements all three.
- Only the `JsonSerializerOptions` argument reaches example and schema generation; the `IJsonTypeInfoResolver` argument of the two-argument constructor is stored but unused, so set `TypeInfoResolver` on the options (as above). Naming policy, converters, and attributes in those options shape the output.
- The parameterless and options-only constructors, and `JsonContentTypeHandler.Shared`, rely on reflection and are annotated `RequiresDynamicCode`/`RequiresUnreferencedCode`; avoid them in trimmed or AOT builds.
- `EnumerationExampleCount` (default `1`) sets how many array items are sampled; `IncludeDescriptionAnnotations` (default `true`) adds `//` comments with descriptions, `Required`, and `Nullable`.
- The generated body example is a type sketch, not a parseable payload: values are placeholders such as `string`, `number`, or `boolean`, arrays end with `...`, and comments are included. Do not parse it as JSON, and use a literal `Example` when a realistic payload matters.
- To support another format, implement the handler interfaces (`GetBodyExampleForType` returns `BodyExampleResult(contents, language)`; `GetParameterExamplesForType` returns `ParameterExampleResult[]`; `GetJsonSchemaForType` returns a `LightJson.Schema.JsonSchema`).

Use `PayloadType` by default. Use literal `Example`, `ExampleLanguage`, or `JsonSchema` only when no representative type exists or generation cannot express the contract.

## Exporters

All implement `IApiDocumentationExporter.ExportDocumentationContent(ApiDocumentation)`, which returns an `HttpContent`; implement it to emit another format.

- `OpenApiExporter` (`Sisk.Documenting.Exporters`) returns OpenAPI JSON. Properties: `OpenApiVersion` (default `"3.0.0"`), `ServerUrls`, `Contact`, `License`, `TermsOfService`. Sisk `<id>` path placeholders become `{id}`; a route with several methods yields one operation per method. Info `title` defaults to "API Documentation" and `version` to `1.0.0` when the context omits them. A route with no `[ApiResponse]` gets a default `200`. It does not emit `JsonSchema` values: request bodies are schematized only from `[ApiParameter]`/`[ApiParametersFrom]` fields, and response and request payloads appear as examples with a generic `object` schema. The media type derives from `ExampleLanguage` (`json`, `xml`, `html`, `text`/`plain`; anything else or empty becomes `application/json`).
- `HtmlDocumentationExporter` (`Sisk.Documenting.Html`) returns a self-contained page with sidebar, search, and Example/Schema tabs, so it does show `JsonSchema`. Customize with `PageTitle`, `Style`, `Script`, `Head`, `Header`, `Footer` (the last three accept renderable objects), `IncludeSidebar` (default `true`), and the `Format*` label properties. `IncludePrismJs()` adds syntax highlighting. Descriptions are rendered as CommonMark. Requesting the same URL with `?embed-target=<endpoint id or name>` returns a script that embeds a single endpoint in another page.
- `LlmsApiExporter` (`Sisk.Documenting.Exporters`) returns a plain-text `llms.txt`-style document including request/response examples and schemas. Properties `Hostname`, `HeaderText`, `FooterText`, also settable through constructors `(hostname)`, `(hostname, headerText)`, and `(hostname, headerText, footerText)`. Without `Hostname` it uses the request's host or relative URLs.

## Required validation

After changing annotations, context, or an exporter:

1. request each documentation endpoint; parse the OpenAPI body as JSON;
2. verify paths, methods, parameters, schemas, and every observable response status;
3. open the HTML (or `llms.txt`) output and inspect grouping, descriptions, examples, schemas, and special characters;
4. compare the outputs with actual serialization, required fields, authentication, and error behavior.

An HTTP 200 from the documentation endpoint is not sufficient; the generated contract must match the executing routes.
