# 12 - MCP Integrations Registration Flow

How backend MCP actions are registered and exposed as tools in the `mcp-integrations` workspace.

## Overview

The `mcp-integrations` workspace defines **code-level** MCP tools that run inside the Backstage backend process. These are not deployed containers -- they are TypeScript functions that call Backstage services (Catalog, Scaffolder, TechDocs) and are exposed to AI clients via the MCP protocol.

## Architecture

```
createXxxAction.ts          -- Define action (name, schema, handler)
    ↓ called from
src/actions/index.ts        -- Aggregator collects all actions
    ↓ called from
src/plugin.ts               -- Backstage plugin with DI deps
    ↓ loaded by
packages/backend/index.ts   -- backend.add(import(...))
    ↓ registered into
ActionsRegistryService      -- In-memory action store
    ↓ exposed by
mcp-actions-backend         -- HTTP at /api/mcp-actions/v1
    ↓ consumed by
AI Client (Cursor, Claude)  -- Streamable HTTP + Bearer token
```

## Registration API

Each action is registered via `actionsRegistry.register()`:

| Field | Type | Purpose |
|-------|------|---------|
| `name` | string | Unique action identifier (e.g., `query-catalog-entities`) |
| `title` | string | Human-readable title |
| `description` | string | LLM-facing description with usage examples |
| `schema.input` | `(z) => ZodObject` | Zod schema factory for input validation |
| `schema.output` | `(z) => ZodObject` | Zod schema factory for output structure |
| `attributes` | object | `{ destructive?, readOnly?, idempotent? }` |
| `visibilityPermission` | BasicPermission | Optional RBAC permission |
| `action` | async function | Handler receiving `{ input, credentials, logger }` |

## Registered Tools (10 total)

### software-catalog-mcp-extras (1 tool)
- `query-catalog-entities` -- Search/list catalog entities with filters

### scaffolder-mcp-extras (6 tools)
- `fetch-template-metadata` -- Get template parameters and steps
- `list-scaffolder-tasks` -- List task history with pagination
- `get-scaffolder-task-logs` -- Get task execution logs
- `list-scaffolder-actions` -- List available scaffolder actions
- `validate-scaffolder` -- Dry-run template validation
- `execute-template` -- Execute a scaffolder template

### techdocs-mcp-extras (3 tools)
- `fetch-techdocs` -- List entities with TechDocs
- `analyze-techdocs-coverage` -- TechDocs coverage statistics
- `retrieve-techdocs-content` -- Fetch documentation content

## Configuration

```yaml
backend:
  actions:
    pluginSources:
      - 'software-catalog-mcp-extras'
      - 'techdocs-mcp-extras'
      - 'scaffolder-mcp-extras'
  auth:
    externalAccess:
      - type: static
        options:
          token: ${MCP_TOKEN}
          subject: mcp-clients
```

## Adding New Tools

1. Create `src/actions/createXxxAction.ts` with `actionsRegistry.register({...})`
2. Add test file `createXxxAction.test.ts`
3. Wire into aggregator `src/actions/index.ts`
4. Update `plugin.ts` deps if new services are needed
5. Run `yarn tsc` and `npx backstage-cli package test`

## Porting from Upstream

The `.cursor/rules/port-mcp-tool.mdc` rule documents the process for copying actions from `backstage/backstage` into overlay plugins, including copyright updates and dependency resolution.
