# BYO Agent Flow Diagrams

Flow diagrams for the BYO (Bring Your Own) Agent feature in **Orion** integrated with **Red Hat Agent Operator**.

## How to Use

Each `.mermaid` file can be imported into [Excalidraw](https://excalidraw.com/):

1. Open [excalidraw.com](https://excalidraw.com/)
2. Click the **Mermaid icon** in the toolbar (or Menu > Insert > Mermaid)
3. Paste the contents of any `.mermaid` file
4. Click **Insert**
5. The diagram is now editable in Excalidraw

## Diagrams

| File | Description |
|------|-------------|
| `01-image-deploy.mermaid` | BYO Agent creation via container image (sequence) |
| `02-source-deploy.mermaid` | BYO Agent creation via Git source + Shipwright build (sequence) |
| `03-auth-token-flow.mermaid` | Authentication and token flow from browser to agent (sequence) |
| `04-lifecycle-state-machine.mermaid` | Agent lifecycle: Draft > Pending > Published > Archived (state diagram) |
| `05-architecture.mermaid` | Overall system architecture (component diagram) |
| `06-delete-flow.mermaid` | Agent delete paths and cleanup (flowchart) |

## Naming

- **Orion** = Red Hat Developer Hub (Backstage-based)
- **Red Hat Agent Operator** = Kagenti (agent platform operator)
