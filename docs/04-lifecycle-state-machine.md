# Agent Lifecycle State Machine

This document describes the four lifecycle stages that every agent passes through, the valid transitions between them, and the rules governing each transition.

---

## Overview

All agents in Orion — regardless of type (BYO/Kagenti, orchestration, workflow builder, skills) — share the same governance lifecycle:

```
┌───────┐       ┌─────────┐       ┌───────────┐       ┌──────────┐
│ Draft │──────▶│ Pending │──────▶│ Published │──────▶│ Archived │
└───────┘       └─────────┘       └───────────┘       └──────────┘
    ▲               │                   │                   │
    │               │                   │                   │
    └───────────────┘                   │                   │
        (withdraw)                      │                   │
    ▲                                   │                   │
    │                                   │                   │
    └───────────────────────────────────┘                   │
        (request unpublish → pending → reject)              │
    ▲                                                       │
    │                                                       │
    └───────────────────────────────────────────────────────┘
        (admin reactivate)
```

---

## Stage: Draft

The initial state for all agents.

### Entry Conditions

- Automatically assigned when an agent is created (via BYO wizard, orchestration builder, or workflow builder)
- Assigned during startup reconciliation for auto-registered agents (`createdBy: system:startup`)
- Returned to draft on admin rejection or owner withdrawal from pending
- Returned to draft on admin reactivation from archived

### Properties

| Property | Value |
|----------|-------|
| `lifecycleStage` | `draft` |
| `published` | `false` |
| `visible` | `false` |
| `createdBy` | User who created it, or `system:startup` for auto-registered |

### Visibility

- Visible to the **creator** (owner) only
- Visible to **admins** (in Agent Registry / Command Center)
- **Not visible** in Marketplace or Explore tab

### Allowed Actions

- **Edit**: Display name, description, avatar, greeting message, conversation starters
- **Delete**: Owner can delete own draft agents. Admins can delete any draft.
- **Submit for Review**: Transitions to pending (requires authentication)

---

## Stage: Pending

Agent has been submitted for admin review.

### Entry Conditions

- Transition from draft via `PUT /agents/:id/promote` with `targetStage: pending`
- Transition from published via owner requesting unpublish (`pendingAction: unpublish`)

### Properties

| Property | Value |
|----------|-------|
| `lifecycleStage` | `pending` |
| `pendingAction` | `publish` or `unpublish` (tracks what the pending review is for) |
| `approvalWorkflowInstanceId` | SonataFlow instance ID (if workflow governance enabled) |

### Rules

- **Any authenticated user** can submit their own agents for review
- **Self-approval blocked**: The admin who submitted the agent cannot approve their own agent
  - Exception: `augment.agentApproval.allowSelfApproval=true` in config overrides this
- **SonataFlow integration** (if enabled):
  - Workflow instance started on transition to pending
  - Notifications sent to admins
  - 72-hour escalation timer begins

### Allowed Actions

- **Admin Approve**: Transitions to published
- **Admin Reject**: Transitions back to draft (with reason)
- **Owner Withdraw**: Transitions back to draft, cancels any active workflow

---

## Stage: Published

Agent is live and visible to all users.

### Entry Conditions

- Transition from pending via admin approval
- Callback from SonataFlow after approval decision

### Properties

| Property | Value |
|----------|-------|
| `lifecycleStage` | `published` |
| `published` | `true` |
| `visible` | `true` |
| `promotedAt` | Timestamp of publication |
| `promotedBy` | Admin who approved |

### Visibility

- Listed in **Marketplace** (Explore tab)
- Visible to **all authenticated users**
- Available for **chat** by all users
- Can be **featured** on the welcome screen (maximum 4 featured agents)

### Allowed Actions

- **Admin Archive**: Directly transitions to archived
- **Owner Request Unpublish**: Transitions to pending with `pendingAction=unpublish`

---

## Stage: Archived

Agent removed from active use, preserved for audit history.

### Entry Conditions

- Transition from published via admin archive action

### Properties

| Property | Value |
|----------|-------|
| `lifecycleStage` | `archived` |
| `published` | `false` |
| `visible` | `false` |

### Characteristics

- **Removed** from Marketplace — no longer discoverable
- **Preserved** for audit history — record still exists in admin config
- **Kubernetes workload may still be running** — archiving is a governance action, not a teardown
- **Chat disabled** for all users

### Allowed Actions

- **Admin Reactivate**: Transitions back to draft (agent re-enters lifecycle from the beginning)

---

## Invalid Transitions

The following transitions are **blocked** by the `isValidTransition()` function:

| From | To | Why |
|------|----|-----|
| Draft | Published | Must go through pending review |
| Draft | Archived | Cannot archive something never published |
| Archived | Pending | Must reactivate to draft first |
| Archived | Published | Must reactivate to draft first, then re-submit |
| Pending | Archived | Must be published before archiving, or rejected back to draft |

Attempting an invalid transition returns a 400 error with a message indicating the transition is not allowed.

---

## Transition Enforcement

The lifecycle is enforced by the `LIFECYCLE_TRANSITIONS` constant defined in `augment-common`. This constant maps each stage to its allowed target stages:

```
LIFECYCLE_TRANSITIONS = {
  draft: [pending],
  pending: [draft, published],
  published: [pending, archived],
  archived: [draft]
}
```

---

## Transition Application

The `applyLifecycleTransition()` function handles every transition:

1. **Validates** the transition is legal (source stage → target stage exists in `LIFECYCLE_TRANSITIONS`)
2. **Updates** the `chatAgents` config entry with new `lifecycleStage`
3. **Increments** the `version` number (optimistic concurrency)
4. **Sets metadata**:
   - `promotedAt`: timestamp of the transition
   - `promotedBy`: userRef of the actor (admin or workflow callback)
5. **Logs audit event**: records who did what and when
6. **Triggers side effects** (if applicable):
   - Starts SonataFlow workflow on draft→pending (if enabled)
   - Sends notifications
   - Clears `pendingAction` and `approvalWorkflowInstanceId` on completion

---

## Permission Model per Transition

| Transition | Who Can Do It |
|-----------|---------------|
| Draft → Pending | Owner (any authenticated user for their own agent) |
| Pending → Published | Admin (not the submitter, unless self-approval enabled) |
| Pending → Draft (reject) | Admin |
| Pending → Draft (withdraw) | Owner |
| Published → Archived | Admin |
| Published → Pending (unpublish request) | Owner |
| Archived → Draft | Admin |

---

## Lifecycle Applies to All Agent Types

This governance lifecycle is universal across:

- **BYO/Kagenti agents** (deployed via wizard)
- **Orchestration agents** (built with multi-agent orchestration tools)
- **Workflow Builder agents** (created with visual workflow editor)
- **Skill agents** (single-skill agents deployed via DevSpaces)

The same `agentRoutes.ts` endpoints handle lifecycle for all types.
