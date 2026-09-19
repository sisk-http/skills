# Sisk Agent Skills

Agent guidance for developing applications with [Sisk](https://github.com/sisk-http). This catalog contains 15 skills covering the framework, Cadente, MCP servers, endpoint documentation, and monitoring.

## Installation

With [Node.js](https://nodejs.org/) and `npx` available, install the skills with:

```bash
npx skills add sisk-http/skills
```

### Manual installation

Clone the repository:

```bash
git clone https://github.com/sisk-http/skills.git
```

For Avi, copy the contents of this repository's `skills/` directory into `$HOME/.agents/skills/` for global availability, or into your project's `.agents/skills/` directory. Preserve the nested directories: the entrypoints link to focused skills using relative paths. Review existing files before copying to avoid overwriting local customizations.

Other agents can read the Markdown files directly or use their supported skill installation mechanism. Discovery and installation behavior depend on the agent.

## Catalog

| Area | Entrypoint | Focused skills |
| --- | --- | --- |
| Framework | [sisk-framework](skills/sisk-framework/SKILL.md) | [Routing](skills/sisk-framework/sisk-framework-routing/SKILL.md), [HTTP](skills/sisk-framework/sisk-framework-http/SKILL.md), [Handlers](skills/sisk-framework/sisk-framework-handlers/SKILL.md), [Realtime and logging](skills/sisk-framework/sisk-framework-realtime/SKILL.md) |
| Cadente | [sisk-cadente](skills/sisk-cadente/SKILL.md) | [Setup](skills/sisk-cadente/sisk-cadente-setup/SKILL.md), [Host APIs](skills/sisk-cadente/sisk-cadente-host/SKILL.md), [Reference types](skills/sisk-cadente/sisk-cadente-reference/SKILL.md) |
| Model Context Protocol | [sisk-model-context-protocol](skills/sisk-model-context-protocol/SKILL.md) | [Setup](skills/sisk-model-context-protocol/sisk-mcp-setup/SKILL.md), [Tools and schemas](skills/sisk-model-context-protocol/sisk-mcp-tools-schema/SKILL.md), [Runtime and results](skills/sisk-model-context-protocol/sisk-mcp-runtime/SKILL.md) |
| Endpoint documentation | [sisk-document](skills/sisk-document/SKILL.md) | Annotations, request/response examples, and OpenAPI/HTML generation |
| Monitoring | [sisk-monitoring](skills/sisk-monitoring/SKILL.md) | Dashboard setup, authentication, counters, meters, and log capture |

## Usage

Ask your agent to use the skill matching the task, for example:

```text
Use sisk-framework-routing to add a route to the existing RouterModule.
```

Read a package entrypoint when choosing a topic; read a focused skill directly when the topic is already known. Load only the guidance relevant to the task. Skills provide context, not permission to deploy, expose services, or invoke external tools.

## Compatibility and validation

Check APIs and defaults against the application's installed package version, especially for experimental components such as Cadente and Sisk.ModelContextProtocol. Examples are illustrative and may rely on application-provided types or surrounding setup.

The published catalog was checked for metadata, relative links, preserved skill names, balanced code fences, and selection/safety scenarios. These structural checks do not establish runtime compatibility with every Sisk release; the examples were not compiled or executed as a complete application.

## Contributing

Keep changes focused, preserve public skill names and relative links, and verify version-sensitive claims against the relevant source or official documentation. Do not include credentials, private application data, or machine-specific configuration.

## License

[MIT](LICENSE).
