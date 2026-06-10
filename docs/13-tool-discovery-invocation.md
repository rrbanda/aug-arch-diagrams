# 13 - Tool Discovery and Invocation Flow

How deployed MCP tools are discovered (connected to) and invoked for testing.

## Discovery Flow (Connect)

After a Kagenti tool is deployed and running:

1. **User clicks Connect** on a tool in `KagentiToolsPanel`
2. `ToolConnectDialog` opens and immediately calls `connectTool()`
3. **Frontend:** `api.connectKagentiTool(namespace, name)`
4. **Backend route:** `POST /kagenti/tools/:ns/:name/connect`
5. **Kagenti API:** Connects to the tool pod's MCP endpoint via MCP protocol
6. **Returns:** Array of `KagentiMcpToolSchema[]` -- each with `name`, `description`, `input_schema`
7. **UI renders:** `McpToolCatalog` displays each discovered tool as a card

Each tool card shows:
- Tool name and description
- Input schema properties with types and descriptions
- Required fields highlighted
- "Invoke" button for testing

## Invocation Flow (Test)

Two entry points for invoking a tool:

### Schema-based invocation (from McpToolCatalog)
1. User clicks "Invoke" on a discovered tool card
2. `SchemaInvokeForm` renders form fields from the JSON schema
3. User fills in values using typed form inputs
4. Submit calls `invokeTool(namespace, name, toolName, arguments)`

### Manual invocation (direct)
1. User clicks the Play icon on a tool row
2. `ToolInvokeDialog` opens with free-form fields
3. User enters tool name and JSON arguments manually
4. Submit calls the same `invokeTool` endpoint

### API path
- **Frontend:** `api.invokeKagentiTool(ns, name, toolName, args)`
- **Backend:** `POST /kagenti/tools/:ns/:name/invoke`
- **Body:** `{ tool_name: string, arguments: Record<string, unknown> }`
- **Kagenti:** Proxies MCP tool call to the tool pod
- **Response:** `{ result: unknown }` -- raw tool output

## Detail Drawer

Clicking a tool row opens `KagentiToolDetailDrawer` which loads:
- **Tool detail:** Full metadata, spec, and status from the K8s CRD
- **Route status:** Whether the HTTP route is ready and accessible
- **Build info:** Shipwright build history and status
- **MCP section:** Inline discovery + invoke (same as Connect dialog)

## Error Handling

- **Connection failure:** Shows error with guidance (check tool status, name suffix issues)
- **Invoke failure:** Displays error message from the MCP tool
- **Tool not ready:** Status chip shows current state (Pending, Building, etc.)
