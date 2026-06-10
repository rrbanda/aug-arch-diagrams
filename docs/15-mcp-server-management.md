# 15 - MCP Server Management Flow

How global MCP servers are configured, tested, and used during chat sessions.

## Purpose

Separate from Kagenti-deployed tool workloads, the platform can connect to **external MCP servers** for use during AI chat sessions. These servers provide additional tool capabilities (e.g., database access, CI/CD, monitoring) to the LLM.

## Configuration Sources

MCP server configuration comes from two sources, merged at runtime:

### 1. YAML baseline (`app-config.yaml`)
```yaml
augment:
  mcpServers:
    - id: my-server
      name: My MCP Server
      type: streamable-http
      url: https://mcp.example.com/v1
      headers:
        Authorization: Bearer ${TOKEN}
      allowedTools: [tool1, tool2]
      requireApproval: always
```

### 2. Admin DB overrides (`augment_admin_config` table)
- `mcpServers` key -- admin-added servers or overrides of YAML servers
- `disabledMcpServerIds` key -- YAML server IDs to exclude

### Merge logic (`RuntimeConfigResolver.mergeMcpServers`)
1. Start with YAML servers
2. For each DB entry with matching YAML ID: override url/name/type/headers/allowedTools/requireApproval, preserve YAML-only auth fields
3. Append DB-only entries
4. Remove disabled IDs

## Admin UI Flow

### Adding a server
1. Admin opens Model Tools panel > MCP Servers tab
2. Clicks "Add Server" -- opens `McpServerFormDialog`
3. Fills in: ID, Name, Type (streamable-http/sse), URL, Bearer token
4. Clicks "Test Connection"
5. Backend `McpTestService` connects via MCP SDK, performs SSRF check
6. On success: shows `AllowedToolsSelector` with discovered tools
7. Admin selects which tools to expose, toggles HITL approval
8. Clicks "Add" -- server added to local state
9. Clicks "Save" -- persisted to `AdminConfigService`

### Managing servers
- **Edit:** Re-open form, modify config, re-test
- **Disable YAML server:** Toggle off -- adds ID to `disabledMcpServerIds`
- **Remove admin server:** Delete from admin list
- **Reset:** Clears all DB overrides, reverts to YAML-only

## Health Monitoring

`StatusService` runs health checks periodically:
1. Connects to each effective MCP server via MCP SDK
2. Lists tools from each server
3. Reports `MCPServerStatus`: connected, toolCount, tools[], error, source
4. Exposed via `GET /status` and polled by `useStatus()` hook (every 30s)

## Runtime Tool Resolution

When a chat request is processed:

### Direct mode (default)
- MCP server configs passed to Llama Stack Responses API
- Llama Stack opens MCP connections directly
- Backstage backend does not see tool call traffic

### Backend mode
- `BackendToolExecutor` opens MCP SDK clients per server
- Discovers tools, converts to `type: 'function'` for the LLM
- Intercepts tool calls and executes them server-side
- Tool names are namespaced: `{serverId}__{toolName}`

## Agent-Level Scoping

Individual agents can restrict which MCP servers they use:
- Configured via `agent.mcpServers: string[]` (server IDs)
- Empty array = all configured servers available
- Enforced in backend mode by `buildAgentToolFilter()`

## Authentication

MCP server auth is resolved by `McpAuthService` with priority:
1. `authRef` -- named config from `augment.mcpAuth[]`
2. Inline `oauth` on the server config
3. Inline `serviceAccount`
4. Global `augment.security.mcpOAuth` (full security mode only)

The admin UI only supports Bearer token via headers. OAuth/SPIFFE must be configured in YAML.
