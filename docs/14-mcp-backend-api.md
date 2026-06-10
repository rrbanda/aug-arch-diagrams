# 14 - MCP Backend API Architecture

Complete REST API surface for MCP tool operations.

## API Layers

The API flows through three layers:

1. **Frontend API** (`toolEndpoints.ts`) -- typed functions called by React components
2. **Backend Routes** (`kagentiToolRoutes.ts`, `toolLifecycleRoutes.ts`) -- Express handlers with auth middleware
3. **Kagenti Client** (`KagentiApiClient.ts`) -- HTTP client proxying to the Kagenti Operator API

## Kagenti Tool Routes

Base path: `/kagenti/tools`

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `GET` | `/kagenti/tools` | namespace validation | List tools across visible namespaces |
| `GET` | `/kagenti/tools/:ns/:name` | namespace validation | Get tool detail |
| `POST` | `/kagenti/tools` | none (name/ns required) | Create tool |
| `DELETE` | `/kagenti/tools/:ns/:name` | **admin only** | Delete tool + cleanup |
| `GET` | `/kagenti/tools/:ns/:name/route-status` | namespace validation | HTTP route readiness |
| `GET` | `/kagenti/tools/:ns/:name/build-info` | namespace validation | Shipwright build status |
| `POST` | `/kagenti/tools/:ns/:name/buildrun` | **admin only** | Trigger new build |
| `POST` | `/kagenti/tools/:ns/:name/finalize-build` | **admin only** | Finalize: create deployment from built image |
| `POST` | `/kagenti/tools/:ns/:name/connect` | namespace validation | MCP discovery: list tool schemas |
| `POST` | `/kagenti/tools/:ns/:name/invoke` | namespace validation | MCP invoke: call a specific tool |

## Tool Lifecycle Routes

Base path: `/tools`

These overlay lifecycle governance on top of Kagenti tools:

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| `GET` | `/tools` | user | Unified list: Kagenti tools + lifecycle metadata |
| `PUT` | `/tools/:toolId/promote` | creator or admin | Advance lifecycle stage |
| `PUT` | `/tools/:toolId/demote` | **admin only** | Regress lifecycle stage |
| `PUT` | `/tools/:toolId/publish` | **admin only** | Shortcut to published state |
| `PUT` | `/tools/:toolId/unpublish` | **admin only** | Move back to pending |

`toolId` format: `"namespace/name"`

Lifecycle metadata stored in `AdminConfigService` under the `chatTools` config key.

## Shipwright Build Routes

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/kagenti/tools/shipwright-builds` | List tool-specific builds |
| `GET` | `/kagenti/shipwright/builds` | List all Shipwright builds (agents + tools) |

## No UPDATE Route

There is no `PUT`/`PATCH` endpoint for Kagenti tools. To modify a tool, it must be deleted and recreated.
