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

## Naming

- **Orion** = Developer Hub platform (Backstage-based)
- **Red Hat Agent Operator** = Agent lifecycle operator (K8s)
- **OGX** = LLM / Frontier model hosting layer
