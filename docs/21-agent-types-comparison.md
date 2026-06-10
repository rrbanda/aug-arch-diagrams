# 21 - Agent Types Comparison

Side-by-side comparison of all agent types, their storage, runtime execution, and governance integration.

## Agent Type Matrix

| Dimension | Skills Agent | Workflow Agent | Config Agent | BYO Agent |
|-----------|-------------|----------------|--------------|-----------|
| **Framework label** | `docsclaw` | `workflow-builder` | `responses-api` | varies (a2a, langgraph, etc.) |
| **Source** | `skills` | `orchestration` | `orchestration` | `kagenti` |
| **Created via** | Skills wizard | Visual Canvas | AgentsPanel / YAML | CreateAgentWizard |
| **Storage** | K8s + chatAgents DB | workflows DB + chatAgents | agents DB + chatAgents | K8s CRD + chatAgents |
| **Deploys to cluster** | Yes (platform image) | No | No | Yes (user image) |
| **Chat runtime** | HTTP to pod endpoint | Responses API in-process | Responses API in-process | Kagenti A2A to pod |
| **Multi-agent support** | No (single agent) | Yes (via handoff edges) | Yes (via handoffs/asTools) | No (single pod) |
| **MCP tools** | Via skill mount | Via MCP nodes | Via mcpServers config | Via Kagenti platform |

## Unified Agent Catalog

All agent types appear in the same catalog via `GET /agents`, which merges:
- **Kagenti source** -- K8s deployed agents (BYO + skills with chatEndpoint)
- **Orchestration source** -- Config agents + workflow agents
- **Skills source** -- DocsClaw skill agents

Each agent overlays governance metadata from the `chatAgents` DB key.

## Lifecycle State Machine

All agent types share the same governance lifecycle:

| From | To | Action | Who |
|------|-----|--------|-----|
| draft | pending | Submit for Review | Creator |
| pending | published | Approve | Admin |
| pending | draft | Reject / Withdraw | Admin / Creator |
| published | pending | Request Unpublish | Admin |
| published | archived | Archive | Admin |
| archived | draft | Reactivate | Admin |

## Agent Roles (Auto-Derived)

Roles apply to config and workflow agents with multi-agent topologies:

| Role | Condition | Catalog visibility |
|------|-----------|-------------------|
| **Router** | Has outgoing handoffs or asTools | Visible (entry point) |
| **Specialist** | Target of another agent's edges | Hidden |
| **Standalone** | No connections | Visible |

Roles are computed by `deriveRoleFromTopology()` and never stored directly.

## Chat Routing

When a user selects an agent to chat with:
1. The `model` field in the chat request is set to the agent ID
2. If the ID matches a Kagenti agent (`namespace/name`), chat goes to the **Kagenti provider** (A2A protocol to pod)
3. Otherwise, chat goes to the **orchestration provider** (Responses API with WorkflowHydrator)
4. Config agents are auto-migrated to workflow format at chat time
5. Both paths stream responses back to the UI via SSE
