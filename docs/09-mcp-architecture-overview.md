# 09 - MCP Architecture Overview

System-level view of how MCP (Model Context Protocol) tools are managed across the Orion platform.

## Two MCP Systems

The platform has two distinct MCP tool management systems that serve different purposes:

### 1. Augment Plugin (Kagenti Tools)

**Purpose:** Deploy, manage, and invoke MCP tool servers as Kubernetes workloads.

**Flow:**
1. Admin creates a tool via the `CreateToolWizard` (deploy from container image or Git source)
2. The wizard calls `POST /kagenti/tools` which proxies to the Kagenti Operator API
3. The Operator creates a K8s Deployment + Service + optional Route
4. The MCP tool server runs in a pod, exposing tools via the MCP protocol
5. Users discover tools via `ToolConnectDialog` (MCP connect) and invoke them for testing

### 2. MCP Integrations Workspace

**Purpose:** Register in-process Backstage actions as MCP tools, exposed to AI clients (Cursor, Claude, etc.).

**Flow:**
1. Overlay plugins (catalog, scaffolder, techdocs) register actions at backend startup
2. Each action is registered with `actionsRegistry.register()` providing name, schema, and handler
3. The upstream `@backstage/plugin-mcp-actions-backend` exposes registered actions at `/api/mcp-actions/v1`
4. AI clients connect via Streamable HTTP with Bearer token authentication

### 3. Global MCP Server Config (LlamaStack Chat)

**Purpose:** Configure external MCP server URLs for use during AI chat sessions.

**Flow:**
1. Admin adds MCP server URLs via `McpServersSection` in the admin panel
2. Config is stored in `augment_admin_config` DB table, merged with YAML baseline
3. At chat time, tools from these servers are made available to the LLM

## Components

| Component | Layer | Role |
|-----------|-------|------|
| KagentiToolsPanel | Frontend | Tool list, CRUD, connect, invoke |
| ToolCreateIntentDialog | Frontend | Entry point: Develop vs Deploy |
| CreateToolWizard | Frontend | 3-step deploy wizard |
| McpServersSection | Frontend | Global MCP server configuration |
| kagentiToolRoutes | Backend | REST API for Kagenti tool operations |
| AdminConfigService | Backend | DB storage for MCP server config |
| McpTestService | Backend | MCP connection testing |
| mcp-actions-backend | Backend | Exposes Backstage actions as MCP tools |
| ActionsRegistryService | Backend | In-memory action registration |
| Kagenti Operator | External | K8s tool lifecycle management |
