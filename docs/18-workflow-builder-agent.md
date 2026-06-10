# 18 - Workflow Builder Agent Creation

The Visual Canvas / Workflow Builder is a no-code drag-and-drop editor for creating agent workflows that run on the Responses API orchestration layer.

## Entry Points

- AgentCreateIntentDialog > "Create with Visual Canvas"
- KagentiAgentsPanel > "Agent Builder" button
- Marketplace > Create Agent flow

## Dashboard

`WorkflowDashboard` shows:
- Existing workflow drafts and published workflows
- Template gallery with built-in starter workflows
- "New Agent Workflow" creation (blank or from template)

## Built-in Templates

| Template | Description |
|----------|-------------|
| Simple Q&A Agent | Start -> Agent -> End |
| Smart Router | Start -> Classify -> If/Else -> Specialist agents |
| Research & Summarize | Start -> Researcher -> Summarizer |
| Kubernetes Assistant | Start -> Agent + MCP node |
| Content Moderator | Start -> Agent + Transform + SetState |

## Node Types

| Node | Purpose |
|------|---------|
| Start | Entry point with optional greeting message |
| Agent | LLM agent with instructions, model, tools, capabilities |
| Tool | MCP tool or custom function binding |
| Classify | Intent classification for routing |
| Logic | If/else conditional branching |
| MCP | External MCP server connection |
| Transform | Data transformation |
| SetState | Workflow state mutation |
| Guardrail | Safety and content checks |
| End | Workflow termination |

## Edge Types

| Edge | Semantics |
|------|-----------|
| Handoff | Agent-to-agent delegation |
| Tool Binding | Agent-to-tool association |
| Sequence | Step-by-step flow |
| Conditional | Branching based on conditions |
| Guardrail Binding | Agent-to-guardrail check |

## Publish Flow

1. Preview/test via `PreviewChatPanel`
2. Open `PublishDialog` -- validates workflow structure
3. Validation checks: has Start node, has Agent node, agents have instructions, graph is connected
4. Save: `PUT /workflows/:id`
5. Publish: `POST /workflows/:id/publish`
6. `syncWorkflowToChatAgents()` creates governance entries

## Chat Runtime

Published workflows run via `WorkflowHydrator` which converts the graph into OpenAI Agents SDK `Agent` instances, then executes via `Runner.run()` against Llama Stack `/v1/responses`.
