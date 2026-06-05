# BYO Agent: Image Deploy Flow

This document describes the complete flow for deploying a BYO (Bring Your Own) agent from a pre-built container image through the Orion UI.

---

## 1. Entry Point: New Agent Dialog

The user navigates to the **My Agents** tab and clicks **"+ New Agent"**. This opens the **Intent Dialog**, which presents two cards:

- **Create Agent** — build an agent from scratch using orchestration tools
- **BYO Agent** — import an existing agent that is already packaged as a container

The user selects **BYO Agent**, which launches the `CreateAgentWizard`, a 3-step dialog.

---

## 2. Step 1: Basics

The first step collects fundamental metadata:

| Field | Description |
|-------|-------------|
| **Name** | DNS-1123 validated: lowercase alphanumeric characters and hyphens only, must start/end with alphanumeric, max 63 characters. This becomes the Kubernetes resource name. |
| **Namespace** | Dropdown populated from the Red Hat Agent Operator's `GET /api/v1/namespaces?enabled_only=true`. Only namespaces labeled `kagenti-enabled=true` appear in the list. |
| **Protocol** | Set to **A2A** (Agent-to-Agent). This field is read-only — BYO agents always use A2A. |
| **Framework** | Defaults to **LangGraph**. Other options include CrewAI, ADK, and others. Informational metadata only — does not affect deployment behavior. |

---

## 3. Step 2: Deployment

This step presents a **radio toggle** between two deployment methods:

### Container Image (selected for this flow)

| Field | Required | Description |
|-------|----------|-------------|
| **Container Image URL** | Yes | Full container image reference (e.g., `registry.example.com/org/my-agent:v1.0`) |
| **Image Pull Secret** | No | Name of an existing Kubernetes Secret (type `kubernetes.io/dockerconfigjson`) for private registry authentication |

### Source from Git

This option is **disabled** (greyed out) when Shipwright is not installed on the cluster. See `02-source-deploy.md` for this flow.

---

## 4. Step 3: Runtime

Runtime configuration for the agent workload:

| Field | Description |
|-------|-------------|
| **Workload Type** | `Deployment` (default), `StatefulSet`, or `Job` |
| **Environment Variables** | Key-value pairs with source type: direct value, Kubernetes Secret reference, or ConfigMap reference |
| **Service Ports** | Port mappings for the agent container (default: 8000) |
| **Create HTTP Route** | Toggle to expose the agent externally via an OpenShift Route |
| **Auth Bridge** | Toggle to inject authentication sidecar. Mode selector: `proxy-sidecar`, `envoy-sidecar`, `lite`, or `waypoint` |
| **SPIRE** | Toggle to enable SPIFFE/SPIRE identity for the agent |
| **mTLS Mode** | Mutual TLS enforcement level for inter-service communication |

---

## 5. Submit: Request Construction

When the user clicks **Submit**, the frontend executes `buildRequest()` which maps the wizard's `FormState` into a `KagentiCreateAgentRequest` object. This request is sent as:

```
POST /api/augment/kagenti/agents
Content-Type: application/json
```

The request body contains all wizard fields structured for the Red Hat Agent Operator API.

---

## 6. Backend Processing

The Orion backend handles the request through the following sequence:

### 6.1 Namespace Validation

`validateNamespace()` checks that the requested namespace is in the allow-list (namespaces with `kagenti-enabled=true` label). Rejects with 403 if namespace is not permitted.

### 6.2 Token Acquisition

`KeycloakTokenManager` obtains a service account token:

- Uses `client_credentials` grant type
- Client ID: `kagenti-api` (confidential client in Keycloak)
- Client secret: from `augment.kagenti.keycloak.clientSecret` in app-config
- Token is cached in-memory with a 60-second safety buffer before expiry

### 6.3 Proxy to Operator

The full request body is proxied to the Red Hat Agent Operator:

```
POST /api/v1/agents
Authorization: Bearer <keycloak-jwt>
Content-Type: application/json
```

---

## 7. Red Hat Agent Operator Processing

The Operator receives the request and creates Kubernetes resources:

1. **Deployment** (or StatefulSet/Job based on workload type) — runs the agent container image with configured env vars and ports
2. **Service** — ClusterIP service routing traffic to the agent pod (port 8080 -> container port 8000)
3. **Route / HTTPRoute** — if HTTP route was enabled, creates external access endpoint

### Operator Controller Reconciliation

The Kubernetes controller (watching resources with `kagenti.io/type=agent` label) automatically:

- Creates a **Keycloak audience scope** for the new agent (named `agent-{namespace}-{name}-aud` containing the agent's SPIFFE ID)
- Registers the **agent card** by fetching `/.well-known/agent-card.json` from the running pod once healthy

---

## 8. Governance Auto-Registration

After successful Operator response, the backend automatically registers the agent in the governance system:

- Pushes an entry to `chatAgents` admin config with:
  - `lifecycleStage: draft`
  - `published: false`
  - `visible: false`
  - `createdBy: <userRef>` (the authenticated user who submitted)
  - `source: kagenti`
  - `governanceRegistered: true`

This ensures the agent is tracked in Orion's lifecycle system from the moment of creation.

---

## 9. Response and UI Update

1. The Operator returns a success response to the backend
2. The backend returns the response to the wizard
3. The wizard dialog closes
4. The **My Agents** tab refreshes its agent list
5. The new agent appears with:
   - `source = kagenti`
   - `stage = draft`
   - `governanceRegistered = true`

---

## 10. Agent Detail Page

After creation, clicking the agent opens the **detail page** with tabs:

| Tab | Purpose |
|-----|---------|
| **Overview** | Status, metadata, endpoints, runtime info |
| **Design** | Agent configuration and description editing |
| **Test** | Inline chat interface to interact with the agent |
| **Build** | Build history (relevant for source-deploy agents) |
| **Agent Card** | A2A agent card JSON display |

---

## 11. Testing the Agent

The **Test** tab provides `InlineAgentChat`, which allows immediate interaction:

1. User types a message in the chat interface
2. Frontend sends `POST /api/augment/chat/stream` with `model=<namespace>/<name>`
3. Backend calls `resolveProvider()` which performs a cache lookup (30s TTL) of known Red Hat Agent Operator agent IDs
4. Match found → routes to `KagentiProvider`
5. `KagentiProvider` acquires a fresh Keycloak token (with sufficient lifetime for streaming)
6. Proxies the request to the agent pod via A2A protocol
7. Streams SSE (Server-Sent Events) back to the frontend
8. Chat messages render in real-time in the InlineAgentChat component

---

## Summary

The image deploy flow is the simplest path to getting a BYO agent running in the platform. The agent goes from container image to live, testable endpoint in a single wizard submission. The governance system automatically tracks it as a draft, requiring admin approval before it becomes visible in the Marketplace.
