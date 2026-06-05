# SonataFlow Governance Workflow

This document describes the SonataFlow-based governance workflow for agent lifecycle transitions, including configuration, workflow states, approval/rejection flows, and edge cases.

---

## Overview

When `augment.agentApproval.enabled=true`, lifecycle transitions that would normally happen immediately are instead delegated to a SonataFlow workflow. This adds a formal review process with notifications, escalation timers, and audit trails.

---

## Configuration

```yaml
augment:
  agentApproval:
    enabled: true
    serviceUrl: http://sonataflow.sonataflow-infra.svc:8080
    workflowId: agentApproval
    allowSelfApproval: false  # Default: false
```

| Field | Description |
|-------|-------------|
| `enabled` | Enables/disables workflow governance. When false, transitions happen immediately on admin action. |
| `serviceUrl` | Internal cluster URL of the SonataFlow runtime |
| `workflowId` | The workflow definition ID (matches `agent-approval.sw.yaml`) |
| `allowSelfApproval` | If true, the admin who submitted an agent can also approve it. Default is false (blocked). |

---

## Workflow Definition

The workflow is defined in `agent-approval.sw.yaml` following the **SonataFlow Serverless Workflow specification 0.8**.

---

## Developer Submission Flow

When a developer submits an agent for review:

### Step 1: Developer Initiates

Developer clicks **"Submit for Review"** on a draft agent in the UI.

### Step 2: Backend Processes Transition

```
PUT /api/augment/agents/:id/promote
Body: { targetStage: "pending" }
```

`applyLifecycleTransition()` sets `lifecycleStage=pending` on the `chatAgents` entry.

### Step 3: Workflow Started

`AgentApprovalWorkflowService.startWorkflow()` is called with:

```json
{
  "agentId": "<agent-id>",
  "agentName": "<display-name>",
  "submittedBy": "<user:default/developer>",
  "action": "publish"
}
```

### Step 4: SonataFlow Instance Created

```
POST <serviceUrl>/management/processes/<workflowId>
Content-Type: application/json
```

SonataFlow creates a new workflow instance and returns an instance ID.

### Step 5: Instance ID Stored

The workflow instance ID is stored on the agent's `chatAgents` entry:
- `approvalWorkflowInstanceId`: the SonataFlow instance identifier

### Step 6: Failure Handling (Fail-Closed)

If the workflow fails to start (SonataFlow unreachable, invalid workflow, etc.):
- Agent is **reverted to draft** (fail-closed behavior)
- HTTP 502 returned to the frontend
- Error message indicates workflow service is unavailable
- Agent is NOT left in pending state without an active workflow

---

## SonataFlow Workflow States

### State 1: NotifyReviewSubmitted

**Type**: Operation state

**Action**: Sends a Backstage notification (broadcast) to all users:

> "Agent '{agentName}' has been submitted for review by {submittedBy}"

**Transition**: → WaitForDecision

### State 2: WaitForDecision

**Type**: Callback state (suspends execution)

**Waits for**: CloudEvent of type `io.rhdhorchestrator.agent.approval.decision`

**Timeout**: 72 hours

**On timeout**: → EscalateTimeout

**On event received**: → ProcessDecision

### State 3: EscalateTimeout

**Type**: Operation state

**Action**: Sends escalation notification (severity: high):

> "Agent '{agentName}' has been pending review for 72 hours without a decision"

**Transition**: → WaitForDecision (loops back for another 72-hour cycle)

**Overall workflow timeout**: 7 days. After 7 days without a decision, the workflow terminates (agent remains in pending state, requires manual intervention).

### State 4: ProcessDecision

**Type**: Switch state

**Condition**: Evaluates `adminDecision.approved`
- `true` → ApproveAgent
- `false` → RejectAgent

### State 5: ApproveAgent

**Type**: Operation state

**Actions**:
1. Calls Orion backend:
   ```
   PUT /api/augment/agents/:id/promote
   Headers:
     X-Augment-Workflow-Callback: true
   Body: { targetStage: "published" }
   ```
2. Sends approval notification to submitter:
   > "Your agent '{agentName}' has been approved and published"

**Transition**: End (workflow complete)

### State 6: RejectAgent

**Type**: Operation state

**Actions**:
1. Calls Orion backend:
   ```
   PUT /api/augment/agents/:id/demote
   Headers:
     X-Augment-Workflow-Callback: true
   Body: { targetStage: "draft", reason: "<admin's rejection reason>" }
   ```
2. Sends rejection notification to submitter:
   > "Your agent '{agentName}' has been rejected: {reason}"

**Transition**: End (workflow complete)

---

## Admin Approval Flow

When an admin approves a pending agent:

### Step 1: Admin Action

Admin clicks **"Approve"** on a pending agent (in Agent Registry or Command Center).

### Step 2: Backend Checks Workflow State

The backend checks:
1. Is workflow governance enabled? (`augment.agentApproval.enabled === true`)
2. Does this agent have an `approvalWorkflowInstanceId`?

### Step 3: Decision Sent to SonataFlow

If both conditions are true, the backend sends a **CloudEvent** decision:

```
POST <serviceUrl>/management/processes/<workflowId>/instances/<instanceId>/signal
Content-Type: application/cloudevents+json

{
  "specversion": "1.0",
  "type": "io.rhdhorchestrator.agent.approval.decision",
  "source": "orion-backend",
  "data": {
    "adminDecision": {
      "approved": true,
      "approvedBy": "<user:default/admin>",
      "reason": ""
    }
  }
}
```

### Step 4: Immediate Response to UI

Backend returns to the frontend:

```json
{
  "workflowDecisionSent": true
}
```

The agent remains in **pending** state at this point. The UI may show "Decision submitted, awaiting workflow completion."

### Step 5: SonataFlow Processes Decision

SonataFlow receives the CloudEvent, wakes the workflow from WaitForDecision state, routes to ProcessDecision → ApproveAgent.

### Step 6: Workflow Callback

SonataFlow calls back to Orion:

```
PUT /api/augment/agents/:id/promote
Headers:
  X-Augment-Workflow-Callback: true
Body: { targetStage: "published" }
```

### Step 7: Backend Applies Transition

The backend recognizes `X-Augment-Workflow-Callback: true`:
- **Bypasses** SonataFlow delegation (avoids infinite loop)
- Applies the transition directly: pending → published
- Clears `approvalWorkflowInstanceId`
- Clears `pendingAction`
- Sets `promotedAt` and `promotedBy`

### Step 8: Agent Published

Agent is now in published state, visible in Marketplace, available for chat.

---

## Admin Rejection Flow

Same as approval flow, but with:

```json
{
  "adminDecision": {
    "approved": false,
    "approvedBy": "<user:default/admin>",
    "reason": "Agent does not meet quality standards. Please add error handling."
  }
}
```

Workflow routes to RejectAgent → calls `PUT /agents/:id/demote` with callback header → agent returns to draft.

---

## Self-Approval Prevention

### Rule

If the admin attempting to approve is the **same user** who submitted the agent (`createdBy === userRef`), the approval is **blocked with 403 Forbidden**.

### Configuration Override

```yaml
augment:
  agentApproval:
    allowSelfApproval: true
```

When `allowSelfApproval=true`, self-approval is permitted (useful for small teams or testing).

### Workflow Callback Exception

When the workflow calls back with `X-Augment-Workflow-Callback: true`, the self-approval check is **bypassed**. The workflow is considered a neutral actor — the approval decision was already validated when the admin sent the CloudEvent.

---

## Owner Withdrawal

When an owner withdraws their agent from pending review:

### Steps

1. Owner clicks **"Withdraw"** on their pending agent
2. Backend receives: `PUT /api/augment/agents/:id/withdraw`
3. Backend demotes agent to draft: `lifecycleStage: draft`
4. Clears `pendingAction`
5. **Cancels SonataFlow workflow**:
   ```
   DELETE <serviceUrl>/management/processes/<workflowId>/instances/<instanceId>
   ```
6. Clears `approvalWorkflowInstanceId`

The workflow is terminated immediately — no further notifications or callbacks will occur.

---

## Applies to All Agent Types

This governance workflow applies universally to:

- **BYO/Kagenti agents** (deployed via wizard)
- **Orchestration agents** (built with multi-agent tools)
- **Workflow Builder agents** (created with visual editor)
- **Skill agents** (single-skill deployments)

All agent types use the same `agentRoutes.ts` lifecycle endpoints, and all transitions are subject to the same SonataFlow governance when enabled.

---

## Workflow Disabled Behavior

When `augment.agentApproval.enabled=false` (or not configured):

- Lifecycle transitions happen **immediately** on admin action
- No SonataFlow instance is created
- No notifications are sent via workflow
- No escalation timers
- Admin clicks "Approve" → agent is published instantly
- Self-approval check still applies (enforced in backend regardless of workflow)

---

## Edge Cases and Failure Modes

| Scenario | Behavior |
|----------|----------|
| SonataFlow unreachable on submit | Fail-closed: agent reverted to draft, 502 returned |
| SonataFlow unreachable on approve | CloudEvent delivery fails, 502 returned, agent stays pending |
| Workflow times out (7 days) | Workflow terminates, agent stays pending, requires manual intervention |
| Agent deleted while pending | Known issue: workflow instance NOT cancelled, may send notifications for deleted agent |
| Multiple approvals sent | SonataFlow handles idempotently — only first decision processed |
| Backend restart during pending | No issue — workflow runs independently, callback will still work when backend is available |
| Keycloak token expired during callback | Backend handles normally — acquires fresh token for downstream operations |

---

## Notification Summary

| Event | Recipients | Severity |
|-------|-----------|----------|
| Agent submitted for review | All users (broadcast) | Normal |
| 72-hour escalation | All users (broadcast) | High |
| Agent approved | Submitter | Normal |
| Agent rejected | Submitter | Normal |
