# 19 - Config-Driven Agent Creation

Config-driven agents are defined through the AgentsPanel admin UI or YAML configuration, supporting multi-agent topologies with handoff orchestration.

## Creation Flow

1. **Create Agent** -- `CreateAgentModal` offers preset templates (Blank, Code Assistant, Research Agent, Knowledge Expert)
2. **Configure** -- Multi-tab editor with Instructions, Capabilities, Connections, and Advanced tabs
3. **Save** -- Persists to `AdminConfigService` as the `agents` config key

## Agent Configuration Fields

| Field | Tab | Purpose |
|-------|-----|---------|
| `name` | - | Display name |
| `instructions` | Instructions | System prompt (markdown) |
| `model` | Instructions | LLM model override |
| `mcpServers` | Capabilities | MCP server IDs for tool access |
| `enableRAG` | Capabilities | Enable document retrieval |
| `vectorStoreIds` | Capabilities | Specific vector stores |
| `enableWebSearch` | Capabilities | Web search capability |
| `enableCodeInterpreter` | Capabilities | Code execution |
| `handoffs` | Connections | Agent keys for delegation |
| `asTools` | Connections | Agent keys as callable tools |
| `temperature` | Advanced | LLM temperature |
| `maxOutputTokens` | Advanced | Token limit |
| `toolChoice` | Advanced | auto/required/none |
| `reasoning` | Advanced | Chain-of-thought settings |
| `guardrails` | Advanced | Safety guardrail IDs |

## Multi-Agent Topology

Agents are connected via `handoffs` and `asTools` arrays, forming a directed graph:

- **Router** -- has outgoing handoffs or asTools (entry point for multi-agent)
- **Specialist** -- target of another agent's edges (hidden from catalog)
- **Standalone** -- no connections (independent agent)

Roles are auto-derived by `deriveRoleFromTopology()`, never set manually.

## Storage

- `agents` key -- `Record<string, AgentConfig>` with full agent definitions
- `defaultAgent` key -- entry agent key for multi-agent routing
- `maxAgentTurns` key -- handoff turn limit
- `chatAgents` key -- auto-managed governance entries

YAML baseline (`augment.agents`) is overridden entirely by DB values when present.

## Chat Runtime

1. `RuntimeConfigResolver` merges YAML + DB config
2. `AgentGraphManager` resolves the agent graph with handoff/delegation tools
3. Config agents are auto-migrated to workflow format by `WorkflowMigration`
4. `WorkflowHydrator` converts to OpenAI Agents SDK instances
5. `Runner.run()` executes against Llama Stack
