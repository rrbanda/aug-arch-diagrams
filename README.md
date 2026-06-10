# BYO Agent Flow Diagrams

Flow diagrams for the BYO (Bring Your Own) Agent feature in **Orion** integrated with **Red Hat Agent Operator**.

All diagrams use color coding:
- **Indigo/Blue** = Orion platform (UI, backend)
- **Red** = Security / authentication (Keycloak, middleware)
- **Green** = Red Hat Agent Operator + success states
- **Violet** = Kubernetes resources (pods, services, routes)
- **Amber** = LLM / OGX + warning/draft states
- **Cyan** = Build system (Shipwright, Tekton)

## How to Use in Excalidraw

1. Open [excalidraw.com](https://excalidraw.com/)
2. Click the **Mermaid icon** in the toolbar (or Menu > Insert > Mermaid)
3. Paste the contents of any `.mermaid` file
4. Click **Insert**
5. The diagram renders with colors and is fully editable

## Diagrams

| # | File | Type | Description |
|---|------|------|-------------|
| 1 | `01-image-deploy.mermaid` | Sequence | BYO Agent creation via container image |
| 2 | `02-source-deploy.mermaid` | Sequence | BYO Agent creation via Git source + Shipwright build |
| 3 | `03-auth-token-flow.mermaid` | Sequence | Authentication: browser to agent with Keycloak + SPIFFE |
| 4 | `04-lifecycle-state-machine.mermaid` | Flowchart | Agent lifecycle: Draft > Pending > Published > Archived |
| 5 | `05-architecture.mermaid` | Flowchart | System architecture: all components and data flows |
| 6 | `06-delete-flow.mermaid` | Flowchart | Delete paths: admin vs non-admin, cleanup per surface |
| 7 | `07-security-modes.mermaid` | Flowchart | Security modes: none vs plugin-only vs full RBAC |
| 8 | `08-sonataflow-approval.mermaid` | Sequence | SonataFlow governance: submit, approve/reject, escalation, withdraw |

## Detailed Explanations

Each diagram has a companion doc in `docs/` with step-by-step explanations:

| Doc | Description |
|-----|-------------|
| `docs/01-image-deploy.md` | Every step of image-based BYO agent creation |
| `docs/02-source-deploy.md` | Git source deploy with Shipwright build pipeline |
| `docs/03-auth-token-flow.md` | Complete auth chain: browser to agent pod |
| `docs/04-lifecycle-state-machine.md` | All states, transitions, and permission rules |
| `docs/05-architecture.md` | All components, their roles, and data flows |
| `docs/06-delete-flow.md` | What gets cleaned up from each delete surface |
| `docs/07-security-modes.md` | none vs plugin-only vs full RBAC in detail |
| `docs/08-sonataflow-approval.md` | SonataFlow governance: workflow states, CloudEvents, escalation |

## MCP Tool Diagrams

Diagrams 09–15 cover the MCP (Model Context Protocol) tool management flows.

| # | File | Type | Description |
|---|------|------|-------------|
| 9 | `09-mcp-architecture-overview.mermaid` | Flowchart | MCP system architecture: Kagenti tools, MCP integrations, global MCP servers |
| 10 | `10-tool-creation-user-journey.mermaid` | Flowchart | User journey: all three tool creation paths (Deploy, Template, DevSpace) |
| 11 | `11-tool-deploy-wizard-sequence.mermaid` | Sequence | CreateToolWizard: image deploy vs source deploy with build polling |
| 12 | `12-mcp-integrations-registration.mermaid` | Sequence | Code-level MCP action registration and exposure via mcp-actions-backend |
| 13 | `13-tool-discovery-invocation.mermaid` | Sequence | Tool discovery (MCP connect) and invocation (test) flows |
| 14 | `14-mcp-backend-api.mermaid` | Flowchart | Backend REST API: all tool CRUD routes and data flow layers |
| 15 | `15-mcp-server-management.mermaid` | Sequence | Global MCP server config: add, test, save, and runtime resolution |

| Doc | Description |
|-----|-------------|
| `docs/09-mcp-architecture-overview.md` | All MCP components, their roles, and the three MCP systems |
| `docs/10-tool-creation-user-journey.md` | Every path for creating a new MCP tool |
| `docs/11-tool-deploy-wizard-sequence.md` | CreateToolWizard steps, API request body, image vs source flows |
| `docs/12-mcp-integrations-registration.md` | Code-level action registration, registered tools, adding new tools |
| `docs/13-tool-discovery-invocation.md` | Connect to discover tools, invoke for testing, detail drawer |
| `docs/14-mcp-backend-api.md` | Complete REST API surface with auth requirements |
| `docs/15-mcp-server-management.md` | MCP server config lifecycle: YAML, DB, merge, health, runtime |

## Naming

- **Orion** = Developer Hub platform (Backstage-based)
- **Red Hat Agent Operator** = Agent lifecycle operator (K8s)
- **OGX** = LLM / Frontier model hosting layer
