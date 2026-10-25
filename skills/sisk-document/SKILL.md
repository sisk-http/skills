---
name: sisk-document
description: Use when documenting Sisk HTTP endpoints with Sisk.Documenting, including annotations, JSON examples and schemas, and the OpenAPI, HTML and llms.txt exporters.
---
# Sisk.Documenting

For additional details, see the [official Sisk API Documentation extension guide](https://docs.sisk-framework.org/docs/extensions/api-documentation).

`Sisk.Documenting` (NuGet, prerelease 1.7.0-rc3) generates API documentation from attributes in the `Sisk.Documenting.Annotations` namespace. `HtmlDocumentationExporter` lives in the separate `Sisk.Documenting.Html` package. Install with `dotnet add package <name> --prerelease`.

Documentation is opt-in: a route whose method lacks `[ApiEndpoint]` is not included.

- [Annotations](./references/annotations.md): read when adding or changing `[ApiEndpoint]`, request/response, parameter, header, or parameter-example attributes.
- [Generation and validation](./references/documentation-generation.md): read when calling `UseApiDocumentation`, configuring `ApiGenerationContext`, type handlers, or an exporter, or when checking generated output.
- [Endpoint documentation practices](./references/endpoint-documentation-practices.md): read when deciding what to document, such as authentication via request handlers, response coverage, and hiding endpoints.

Prefer `PayloadType` with the application's real `JsonSerializerOptions`, then validate the generated output against route behavior.
