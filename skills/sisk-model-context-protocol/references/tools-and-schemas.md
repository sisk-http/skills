# Tools and schemas

For additional details, see the [official Sisk Model Context Protocol documentation](https://docs.sisk-framework.org/docs/extensions/mcp).

Supported: **Tools** and static **Skills** (see [Publishing skills](#publishing-skills)). Prompts, Completions, and server-side Logging are not implemented; resources exist only as skill files.

## Define stable tools

Register tools before accepting requests and keep the catalog stable for the provider lifetime. When at least one tool exists, server capabilities include a `tools` object with `listChanged: false`; do not hot-add, remove, or rename tools in response to client activity.

```csharp
var tool = new McpTool(
    name: "search_orders",
    description: "Searches orders belonging to the authenticated tenant.",
    schema: JsonSchema.CreateObjectSchema(
        properties: new Dictionary<string, JsonSchema>
        {
            ["query"] = JsonSchema.CreateStringSchema(description: "Customer-visible search text."),
            ["limit"] = JsonSchema.CreateNumberSchema(description: "Maximum results, from 1 to 50.")
        },
        requiredProperties: ["query"]),
    executionHandler: async ctx =>
    {
        var principal = await RequireCallerAsync(ctx.Request, ctx.Cancellation);
        string query = ctx.Arguments["query"].GetString();
        int limit = ctx.Arguments["limit"].MaybeNull()?.GetInteger() ?? 20;

        if (limit is < 1 or > 50)
            return McpToolResult.CreateText("limit must be between 1 and 50.");

        var orders = await SearchOrdersAsync(principal.TenantId, query, limit, ctx.Cancellation);
        return McpToolResult.CreateText(FormatOrders(orders));
    },
    title: "Search orders");

tool.Annotations = new McpToolAnnotations
{
    ReadOnlyHint = true,
    OpenWorldHint = false
};

provider.Tools.Add(tool);
```

`McpTool` exposes `Name`, `Title`, `Description`, input `Schema`, mutable `ExecuteAsync`, optional `OutputSchema`, `Icons`, and `Annotations`. `tools/list` uses `Name` as the title fallback. Add icons with `McpProviderIcon(sourceUrl, mimeType, sizes, theme)` only when their URLs and media metadata are safe to publish. Use durable names and descriptions that explain purpose, caller constraints, input limits, and side effects. Annotation hints describe behavior; they do not enforce it.

## Input and output schemas

Use `LightJson.Schema.JsonSchema` to publish tool contracts. The provider validates `params.arguments` before invoking `ExecuteAsync`; invalid input returns a result with `isError: true` listing each `[path] message`, and the handler does not run. Missing `arguments` is treated as `{}`. Use `JsonSchema.Empty` for a no-argument tool.

```csharp
JsonSchema inputSchema = JsonSchema.CreateObjectSchema(
    properties: new Dictionary<string, JsonSchema>
    {
        ["status"] = JsonSchema.CreateStringSchema(
            enums: ["open", "closed"],
            description: "Order status."),
        ["tags"] = JsonSchema.CreateArraySchema(
            itemsSchema: JsonSchema.CreateStringSchema(),
            minItems: 1,
            description: "At least one tag.")
    },
    requiredProperties: ["status"]);

var outputSchema = JsonSchema.CreateObjectSchema(
    properties: new Dictionary<string, JsonSchema>
    {
        ["count"] = JsonSchema.CreateNumberSchema(),
        ["items"] = JsonSchema.CreateArraySchema(JsonSchema.CreateStringSchema())
    },
    requiredProperties: ["count", "items"]);
```

Set `tool.OutputSchema` only when the handler returns matching `McpToolResult.StructuredContent` (see [Results](./runtime-and-results.md#results)). With an `OutputSchema`, missing or non-conforming structured content becomes a generic `isError: true` result and the content is not returned. Without one, any `StructuredContent` is sent unvalidated.

A schema is a shape check, **not** an authorization, tenant isolation, rate/concurrency limit, business-rule, or semantic validation mechanism. Independently authenticate the request, authorize the action for the derived tenant and resource, bound sizes and pagination, rate-limit expensive or mutating operations, and validate relationships, state transitions, ownership, and values that JSON Schema cannot establish.

## AOT and trimming

Provider envelopes use the source-generated `McpSerializerContext`. For Native AOT or trimmed deployment, build schemas explicitly with the `JsonSchema.Create*Schema` factories and avoid `JsonSchema.CreateFromType` and `JsonSchema.CreateFromDelegate`, which generate schemas through reflection. Verify `tools/list` and `tools/call` against the published artifact.

## Publishing skills

Add `McpSkill` objects to `provider.Skills` before handling requests; the catalog is static and held in memory (nothing is read from disk by the provider, and supporting bytes are copied at construction). Do not mutate it while serving.

```csharp
provider.Skills.Add(new McpSkill(
    path: "acme/refunds",
    frontmatter: new JsonObject
    {
        ["name"] = "refunds",
        ["description"] = "Follow the refund workflow."
    },
    body: "# Refunds\nRead references/policy.md before using the refund tools.",
    files: new Dictionary<string, byte[]>
    {
        ["references/policy.md"] = File.ReadAllBytes("skills/refunds/references/policy.md")
    }));
```

- The constructor throws `ArgumentException` unless `frontmatter.name` is lowercase letters/digits separated by single hyphens (max 64), `description` is 1–1024 characters, every `path` segment matches `[a-zA-Z0-9._~-]+` (not `.`/`..`), and the **last** `path` segment equals `name`. The URI becomes `skill://{path}/SKILL.md`.
- `body` must not include its own frontmatter; `SKILL.md` is generated as JSON-form frontmatter plus the body. Existing YAML files are not parsed.
- `files` keys are relative `/` paths (no empty, `.`, `..`, control characters, `\`, or `:`) and cannot replace `SKILL.md`. A skill is limited to 512 files and 16 MiB including `SKILL.md`.
- `capabilities.resources` and `capabilities.extensions["io.modelcontextprotocol/skills"]` are advertised only to modern requests (`server/discover`) and only when `Skills` is non-empty. Skill methods are unavailable on the legacy flow.
- Clients use `skills/list`, `skills/get` (by `uri`), `resources/list`, and `resources/read` (by `uri`). `SKILL.md` is returned as text; supporting files are base64 blobs. Manifests carry each file's size and SHA-256 digest. Listings are never paginated: a `cursor` is rejected.
- A nested skill must have every one of its files also registered in the enclosing skill's `files` with identical bytes. Duplicate skill URIs, divergent shared content, or incomplete enclosing manifests make all skill and resource requests return `-32603`.
