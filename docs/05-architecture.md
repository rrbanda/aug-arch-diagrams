# System Architecture

This document describes the major system components, their responsibilities, and how they interact in the BYO agent platform.

---

## Component Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Browser                                   │
│   Orion Frontend (Augment Plugin - React SPA)                    │
└──────────────────────────────┬───────────────────────────────────┘
                               │ REST API + Session Cookie
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Orion Backend                                  │
│   (augment-backend Express.js plugin)                            │
└────────┬──────────────────┬──────────────────┬──────────────────┘
         │                  │                  │
         ▼                  ▼                  ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────────────────┐
│   Keycloak   │   │  Red Hat     │   │   Backstage Database     │
│   (Realm)    │   │  Agent       │   │   (augment_admin_config) │
│              │   │  Operator    │   │                          │
└──────────────┘   └──────┬───────┘   └──────────────────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
       ┌───────────┐ ┌────────┐ ┌──────────┐
       │  Agent    │ │Service │ │  Route   │
       │  Pods     │ │        │ │          │
       └─────┬─────┘ └────────┘ └──────────┘
             │
             ▼
       ┌───────────────────────┐
       │  Hosted LLM /         │
       │  Frontier Models      │
       │  via OGX              │
       └───────────────────────┘
```

---

## Orion Frontend (Augment Plugin)

A React single-page application built with Material UI (MUI) components, running within the Backstage/RHDH shell.

### Key UI Components

| Component | Responsibility |
|-----------|---------------|
| `CreateAgentWizard` | BYO agent creation (3-4 step wizard) |
| `AgentGrid` | Marketplace grid layout with agent cards |
| `CompactAgentCard` | Individual agent card (My Agents view) |
| `AgentLifecycleDetail` | Full agent detail page with tabs |
| `AgentRegistryPanel` | Admin panel for managing all agents |
| `InlineAgentChat` | Chat interface for testing agents |
| `IntentDialog` | Initial agent type selection (Create vs BYO) |

### Communication

- REST API calls to Orion backend
- Includes `X-Backstage-Request` header (plugin identification)
- Backstage session cookie for authentication
- SSE (Server-Sent Events) for streaming chat responses

---

## Orion Backend (augment-backend plugin)

Express.js-based backend plugin that handles all business logic, security enforcement, and external service communication.

### Route Groups

| Route Group | Endpoints |
|-------------|-----------|
| `chatRoutes` | `POST /chat/stream`, `POST /chat/complete` — chat with agents |
| `agentRoutes` | CRUD for governance lifecycle, promote/demote/withdraw |
| `kagentiAgentRoutes` | BYO agent create/delete/build-info/finalize, proxied to Operator |
| `kagentiRoutes` | General Operator queries (namespaces, build strategies, agent list) |
| `skillsRoutes` | Skill agent management |
| `workflowRoutes` | SonataFlow integration, approval workflow management |

### Security Middleware

| Middleware | Purpose |
|-----------|---------|
| `requirePluginAccess` | Auth gate — verifies user has access to the augment plugin |
| `requireAdminAccess` | Admin gate — verifies user has admin privileges |
| `getUserRef` | Identity resolution — extracts user entity reference from session |
| `checkIsAdmin` | Utility — returns boolean admin status without blocking |

### AdminConfigService

A key-value store backed by the `augment_admin_config` database table. Stores:

| Config Key | Content |
|-----------|---------|
| `chatAgents` | Governance registry — all agents with lifecycle state, ownership, metadata |
| `agents` | Orchestration configuration — multi-agent routing rules |
| `workflows` | Workflow Builder definitions and versions |
| `branding` | UI customization (logo, colors, welcome message) |
| `systemPrompt` | Global system prompt prepended to all agent conversations |
| `mcpServers` | MCP server configurations available to orchestrated agents |

### KagentiProvider

The provider responsible for routing requests to BYO agents:

- Uses `KagentiApiClient` for HTTP communication with the Operator
- Uses `KeycloakTokenManager` for token acquisition and caching
- Uses `AsyncLocalStorage` for per-request user context isolation
- Implements streaming SSE proxy for chat

### Provider Resolution

`resolveProvider()` determines which provider handles a given model/agent ID:

1. Maintains a cache of known Red Hat Agent Operator agent IDs (refreshed every 30 seconds)
2. If the requested model matches a Kagenti agent → routes to `KagentiProvider`
3. Otherwise → falls back to orchestration provider (built-in agents, workflow agents, skill agents)

---

## Keycloak

Identity and access management server providing OAuth2/OIDC services.

### Realm Configuration

The agent realm contains:

| Client | Type | Purpose |
|--------|------|---------|
| `kagenti` | Public | UI login flows (authorization code + PKCE) |
| `kagenti-api` | Confidential | Backend service account (client_credentials grant) |
| `rhdh` | Confidential | Orion/Backstage integration |

### Roles

| Role | Assigned To |
|------|-------------|
| `admin` | Service accounts, platform administrators |
| `developer` | Standard users creating and managing agents |
| `ns-admin` | Namespace-scoped administrators |

### Client Scopes

| Scope | Purpose |
|-------|---------|
| `agent-{ns}-{name}-aud` | Per-agent audience scope containing the agent's SPIFFE ID. Auto-managed by the Operator controller. |
| `kagenti-platform-audience` | Platform-wide audience scope for non-agent API calls |

Audience scopes are added as **default client scopes** on the `kagenti-api` client, ensuring all tokens automatically include audience claims for all registered agents.

---

## Red Hat Agent Operator

A Kubernetes operator managing the lifecycle of AI agent workloads.

### API Component (Python FastAPI)

| Endpoint Group | Operations |
|----------------|-----------|
| Agents | CRUD, list, get agent card, chat proxy |
| Build | Build strategies, build-info, finalize-build |
| Namespaces | List enabled namespaces |
| Auth | Auth configuration, token info |
| Tools | MCP tool management |

### Controller Component (Go)

A Kubernetes controller with webhook and reconciler:

| Function | Mechanism |
|----------|-----------|
| Watch agent deployments | Label selector: `kagenti.io/type=agent` |
| Manage agent cards | Fetches `/.well-known/agent-card.json` from agent pods |
| Create Keycloak scopes | Auto-creates audience scope per agent on deploy |
| Inject sidecars | Auth bridge, SPIRE agent based on agent configuration |
| Reconcile state | Ensures desired state matches actual state on the cluster |

### Agent Card Cache

The controller maintains an in-memory cache of agent cards:
- Periodically fetches `/.well-known/agent-card.json` from each running agent pod
- Provides agent capabilities, skills, and metadata to the Orion backend
- Used for Marketplace display and agent discovery

---

## Shipwright Build System

Handles building container images from source code on-cluster.

### Components

| Component | Role |
|-----------|------|
| **Build Controller** | Manages Shipwright `Build` and `BuildRun` CRDs |
| **Tekton Pipelines** | Executes `TaskRun` resources created by Shipwright |
| **Build Strategies** | Define how images are built (e.g., buildah with specific options) |

### Build Strategies

| Strategy | Use Case |
|----------|----------|
| `buildah` | External registries with valid TLS certificates |
| `buildah-insecure-push` | Internal registries without valid TLS (e.g., in-cluster registry) |

### Build Flow

1. Shipwright `Build` resource defines source + output
2. `BuildRun` triggers execution
3. Tekton `TaskRun` created by Shipwright controller
4. Build pod: clones Git repo → executes Dockerfile via buildah → pushes image
5. BuildRun status updated with phase and result

---

## Agent Workloads

The actual agent containers running on the cluster.

### Kubernetes Resources

| Resource | Configuration |
|----------|---------------|
| **Deployment** | `kagenti.io/type=agent` label, container on port 8000 |
| **Service** | ClusterIP, port 8080 → target port 8000 |
| **Route/HTTPRoute** | External HTTPS access (optional) |

### A2A Protocol

Each agent implements the Agent-to-Agent protocol:

| Endpoint | Purpose |
|----------|---------|
| `/.well-known/agent-card.json` | Agent capabilities and metadata |
| `/send` | Synchronous message send |
| `/stream` | Server-sent events streaming |

### Health Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/healthz` | Liveness probe — is the process running? |
| `/readyz` | Readiness probe — is the agent ready to accept traffic? |

---

## LLM Backend (Hosted LLM / Frontier Models via OGX)

The inference layer that agents use for language model capabilities.

### Key Characteristics

- Provides an **OpenAI-compatible API** for inference
- Agents connect via the `LLM_API_BASE` environment variable (set during deployment)
- **No direct connection** from Orion to the LLM — agents handle their own LLM calls independently
- Orion has no visibility into which models agents use or how they prompt them
- This decoupling means agents are fully autonomous in their LLM usage

---

## Data Flow Summary

| Flow | Path |
|------|------|
| Agent creation | Browser → Orion Backend → Operator → K8s API → Resources created |
| Agent chat | Browser → Orion Backend → Operator → Auth Bridge → Agent Pod |
| Agent build | Browser → Orion Backend → Operator → Shipwright → Tekton → Registry |
| Token acquisition | Orion Backend → Keycloak → JWT returned and cached |
| Agent card fetch | Operator Controller → Agent Pod → Card cached |
| LLM inference | Agent Pod → OGX → Response to agent |
| Lifecycle change | Browser → Orion Backend → AdminConfigService → DB |
