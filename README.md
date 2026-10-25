# Sisk Agent Skills

A public catalog of agent skills for building and operating applications with the [Sisk Framework](https://github.com/sisk-http). Each skill represents a Sisk package or a distinct package-level capability and links to focused references loaded only when needed.

## Install

### Prerequisites

- [Node.js](https://nodejs.org/) with `npm` and `npx` available.

### Add the catalog

```bash
npx skills add sisk-http/skills
```

The installer lets you select the skills and supported agents available in your environment. Run the same command again to update an existing installation.

## Available skills

| Skill | Scope |
| --- | --- |
| `sisk-framework` | Core hosting, routing, controller-like `RouterModule` architecture, request-scoped dependencies, handlers, bootstrappers, deferred actions, HTTP, realtime connections, and logging. |
| `sisk-cadente` | Experimental managed HTTP/1.1 engine integration and standalone Cadente host APIs. |
| `sisk-model-context-protocol` | MCP Streamable HTTP setup, tools, schemas, runtime behavior, and protocol errors. |
| `sisk-document` | Endpoint annotations, OpenAPI/HTML generation, schemas, examples, and documentation validation. |
| `sisk-monitoring` | Embedded monitoring dashboards, counters, meters, logs, health history, access control, and persistence. |
| `sisk-ini-configuration` | INI parsing and portable Sisk host configuration for listeners, server settings, CORS, and parameters. |
| `sisk-json-rpc` | JSON-RPC 2.0 methods, HTTP/WebSocket transports, serialization, documentation, and protocol errors. |

## Usage

Ask the agent to use the skill that matches the package or task. For example:

```text
Use the sisk-framework skill to design a RouterModule API with authentication middleware and request-scoped repositories.
```

Every skill starts at `skills/<name>/SKILL.md`. The entrypoint provides decision guidance and links to detailed files under `references/`. References are not separate catalog entries; agents should load only the reference relevant to the current task.

## Scope and source accuracy

The catalog is grounded in current Sisk source, tests, and official documentation. Application architecture examples are intentionally neutral: response envelopes, repository layouts, authorization policy, persistence choices, and business exceptions remain application decisions.

Package APIs can change, especially prerelease or experimental components such as Cadente. Verify the target project's installed Sisk version before applying version-sensitive guidance.

## License

This project is distributed under the [MIT License](LICENSE).
