# Security Modes

This document describes the three security modes available in Orion, configured via `augment.security.mode` in app-config. Each mode controls authentication, authorization, and admin access differently.

---

## Configuration

```yaml
augment:
  security:
    mode: plugin-only  # Options: none, plugin-only, full
    adminUsers:        # Used in plugin-only mode
      - user:default/admin
      - user:default/alice
```

---

## Mode: `none` (Development Only)

**Intended for**: Local development and single-user testing.

### Behavior

| Middleware | Behavior |
|-----------|----------|
| `requirePluginAccess` | **SKIPPED** — all requests are allowed without any authentication check |
| `checkIsAdmin` | Returns **TRUE** for everyone — every user has full admin privileges |
| `getUserRef` | Attempts `httpAuth.credentials()` but **falls back to `user:default/guest`** on any failure. No real identity tracking. |

### Characteristics

- No authentication enforcement whatsoever
- No permission boundaries between users
- No meaningful audit trail (all actions attributed to guest or first available identity)
- Every user can perform every operation: create, publish, delete, archive, manage all agents

### Risks

| Risk | Impact |
|------|--------|
| No auth enforcement | Anyone with network access can call any API |
| No permission boundaries | Any user can delete or modify any other user's agents |
| No audit trail | Cannot trace actions back to responsible users |
| Credential exposure | Admin-only endpoints (like Operator proxy) accessible to all |

### When to Use

- Local development machine with no shared access
- Single-user testing environments
- Quick prototyping where security is not a concern

### When NOT to Use

- **NEVER** in shared development clusters
- **NEVER** in staging environments
- **NEVER** in production
- **NEVER** when multiple users have access to the same instance

---

## Mode: `plugin-only` (Recommended for Shared Environments)

**Intended for**: Shared development clusters and staging environments.

### Behavior

| Middleware | Behavior |
|-----------|----------|
| `requirePluginAccess` | Verifies valid Backstage session cookie. Returns **401 Unauthorized** if no valid session. |
| `checkIsAdmin` | Checks if `userRef` appears in `augment.security.adminUsers` list from app-config |
| `getUserRef` | Resolves from `httpAuth.credentials()`. Returns the actual user entity reference (e.g., `user:default/alice`). |

### Admin Determination

Admins are explicitly listed in app-config:

```yaml
augment:
  security:
    adminUsers:
      - user:default/admin
      - user:default/alice
      - user:default/bob
```

A user is admin if and only if their `userEntityRef` matches an entry in this list.

### Permission Matrix

| Action | Admin | Standard User |
|--------|-------|---------------|
| View published agents in Marketplace | Yes | Yes |
| Chat with published agents | Yes | Yes |
| Create BYO agents (draft) | Yes | Yes |
| Submit own agents for review | Yes | Yes |
| Withdraw own pending agents | Yes | Yes |
| Delete own draft agents | Yes | Yes |
| Approve/reject pending agents | Yes | No |
| Publish/archive agents | Yes | No |
| Delete others' agents | Yes | No |
| Access Command Center | Yes | No |
| Access Agent Registry | Yes | No |
| Manage branding/system prompt | Yes | No |
| Feature agents on welcome screen | Yes | No |

### Characteristics

- Users must be logged into Backstage to access any augment functionality
- Session cookie validates through Backstage's auth system
- Admin list is static (requires app-config change and restart to modify)
- Good balance of security and operational simplicity

---

## Mode: `full` (Production)

**Intended for**: Production Red Hat Developer Hub deployments with RBAC enabled.

### Behavior

| Middleware | Behavior |
|-----------|----------|
| `requirePluginAccess` | Checks `augment.access` permission via Backstage permissions API. Denied → 403 Forbidden. |
| `checkIsAdmin` | Checks `augment.admin` permission via Backstage RBAC. Evaluated by policy engine with roles and conditions. |
| `getUserRef` | Same as plugin-only: resolves from `httpAuth.credentials()`. Returns actual user entity reference. |

### Permission Definitions

| Permission | Purpose | Checked By |
|-----------|---------|------------|
| `augment.access` | Can the user access the augment plugin at all? | Backstage permissions API |
| `augment.admin` | Does the user have admin privileges? | Backstage RBAC policy engine |

### Integration with Backstage Permission Framework

In full mode, permissions are evaluated by the Backstage permission framework:

1. Request arrives at augment backend
2. Middleware calls `permissions.authorize([{ permission: augment.access }])`
3. Backstage permission framework evaluates against configured policies
4. Policies can use:
   - **Roles**: Groups of permissions assigned to users or groups
   - **Conditions**: Dynamic rules based on request context
   - **Catalog ownership**: Permissions based on entity ownership in the catalog

### Characteristics

- Fine-grained, policy-driven access control
- Admin status determined dynamically (not from a static list)
- Can leverage Backstage groups and team structures
- Supports complex authorization scenarios (e.g., namespace-scoped admin)
- Requires RBAC plugin to be installed and configured in RHDH

---

## Comparison Table

| Aspect | `none` | `plugin-only` | `full` |
|--------|--------|---------------|--------|
| Authentication | None | Backstage session | Backstage session |
| Authorization | None (all allowed) | Static admin list | RBAC policy engine |
| Admin determination | Everyone is admin | Config list match | Permission evaluation |
| Identity tracking | Fallback to guest | Real user identity | Real user identity |
| Audit quality | Poor | Good | Best |
| Setup complexity | None | Low (list admins) | Medium (configure policies) |
| Appropriate for | Dev laptop | Shared dev/staging | Production |

---

## Security Middleware Execution Order

For every request to augment backend routes:

```
1. requirePluginAccess(req, res, next)
   ├── mode=none: skip, call next()
   ├── mode=plugin-only: verify session cookie
   └── mode=full: check augment.access permission

2. getUserRef(req)
   ├── mode=none: try auth, fallback to user:default/guest
   └── mode=plugin-only/full: resolve from httpAuth credentials

3. [Route-specific middleware]
   ├── requireAdminAccess: blocks non-admins with 403
   └── checkIsAdmin: returns boolean (non-blocking)
```

---

## Changing Modes

Mode is set in `app-config.yaml` (or environment-specific overlay):

```yaml
# Development
augment:
  security:
    mode: none

# Staging
augment:
  security:
    mode: plugin-only
    adminUsers:
      - user:default/admin
      - user:default/lead-dev

# Production
augment:
  security:
    mode: full
```

Changes require a restart of the Backstage/RHDH backend to take effect. There is no runtime mode switching.
