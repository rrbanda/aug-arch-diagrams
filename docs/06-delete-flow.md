# Agent Delete Flow

This document describes how agents are deleted from the platform, including the three UI surfaces, the smart delete pattern, permission model, and cleanup behavior.

---

## Delete Surfaces in the UI

There are three places in the Orion UI where users can initiate agent deletion. All three now use the same **smart delete** pattern:

### 1. My Agents (CompactAgentCard)

- Trash icon on the agent card
- Click → confirmation dialog ("Are you sure you want to delete this agent?")
- Confirm → smart delete executed

### 2. Agent Detail (AgentLifecycleDetail)

- Delete button on the agent detail page
- Click → confirmation dialog
- Confirm → smart delete executed

### 3. Agent Registry (AgentRegistryPanel)

- Delete button in the admin registry table
- Click → confirmation dialog
- Confirm → smart delete executed

---

## Smart Delete Pattern (BYO/Kagenti Agents)

The smart delete pattern attempts full cleanup and falls back gracefully based on permissions:

### Step 1: Attempt Full Delete

```
DELETE /api/augment/kagenti/agents/:namespace/:name
```

This is an **admin-only** endpoint that proxies to the Red Hat Agent Operator.

### Step 2a: Success (HTTP 200)

If the user has admin access:

1. **Operator deletes Kubernetes resources**:
   - Deployment (or StatefulSet/Job)
   - Service
   - Route/HTTPRoute
   - AgentRuntime custom resource

2. **Backend removes governance entry**:
   - Removes the agent from `chatAgents` admin config
   - Full cleanup complete

Result: Agent is completely removed from both the cluster and the governance system.

### Step 2b: Forbidden (HTTP 403)

If the user is **not an admin**, the full delete fails. The system falls back:

```
DELETE /api/augment/agents/:agentId
```

This governance-only delete:
- Removes the `chatAgents` config entry (agent no longer tracked in Orion lifecycle)
- **Kubernetes resources REMAIN** (Deployment, Service, Route still running)
- Admin must manually clean up K8s resources separately

---

## Non-Kagenti Agent Deletion

For agents that are not BYO/Kagenti (orchestration, workflow, skills), the delete is handled entirely by:

```
DELETE /api/augment/agents/:agentId
```

This endpoint performs **cascading cleanup** based on agent type:

### Orchestration Agents

- Removes entry from `agents` admin config (orchestration routing rules)
- Removes entry from `chatAgents` admin config (governance)

### Workflow Builder Agents

- Removes workflow definition and all versions
- Removes `chatAgents` entry

### Skill Agents

- Removes `chatAgents` entry
- If `chatEndpoint` is set: calls DevSpaces API to tear down K8s resources (workspace pod, service, route)

---

## Permission Model

| User Role | Agent Ownership | Can Delete? | Cleanup Scope |
|-----------|----------------|-------------|---------------|
| Admin | Any | Yes (any stage) | Full: K8s + governance |
| Non-admin | Owner (draft only) | Yes | Governance only (K8s remains for Kagenti) |
| Non-admin | Owner (non-draft) | No | — |
| Non-admin | Not owner | No | — |

### Key Rules

- **Admins** can delete any agent at any lifecycle stage — they get full Operator-level cleanup
- **Non-admin owners** can only delete their own agents that are in **draft** stage
- **Non-admin non-owners** cannot delete agents at all
- Published or pending agents cannot be deleted by non-admins (must be archived first by admin, or withdraw from pending then delete draft)

---

## What Gets Cleaned Up

### Full Delete (Admin)

| Resource | Cleaned Up? |
|----------|-------------|
| K8s Deployment/StatefulSet/Job | Yes |
| K8s Service | Yes |
| K8s Route/HTTPRoute | Yes |
| AgentRuntime CR | Yes |
| chatAgents config entry | Yes |
| agents config entry (if orchestration) | Yes |
| Workflow definitions (if workflow) | Yes |

### Governance-Only Delete (Non-Admin Fallback)

| Resource | Cleaned Up? |
|----------|-------------|
| chatAgents config entry | Yes |
| K8s Deployment/StatefulSet/Job | **No** |
| K8s Service | **No** |
| K8s Route/HTTPRoute | **No** |
| AgentRuntime CR | **No** |

---

## What is NOT Cleaned Up on Delete

Regardless of who performs the delete, the following resources are **not** cleaned up:

| Resource | Reason |
|----------|--------|
| **Shipwright Build resources** | Build and BuildRun CRDs remain in the namespace. Depends on Operator upstream behavior; not managed by Orion delete flow. |
| **Shipwright BuildRun resources** | Same as above — historical build records persist. |
| **Keycloak audience scopes** | `agent-{ns}-{name}-aud` scope is not removed. The Operator controller creates scopes but does not delete them on agent removal. Tokens will still include stale audience claims until scopes are manually removed. |
| **SonataFlow workflow instances** | If an agent is deleted while in pending state with an active approval workflow, the workflow instance is **not cancelled**. This is a known issue — the workflow may eventually time out or send notifications for a non-existent agent. |

---

## Delete Confirmation Dialog

The confirmation dialog shown to users includes:

- Agent name and namespace
- Warning that this action cannot be undone
- For admins: note that Kubernetes resources will also be removed
- For non-admins (draft owners): note that only governance tracking will be removed and K8s resources remain

---

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Operator unreachable | 502 returned, agent not deleted |
| Agent not found in Operator | 404 returned. If governance entry exists, it's cleaned up anyway (stale entry). |
| Governance entry missing | No error — idempotent removal |
| K8s resource already deleted | Operator handles gracefully (no error) |
| Concurrent delete | First succeeds, second gets 404 (handled gracefully) |
