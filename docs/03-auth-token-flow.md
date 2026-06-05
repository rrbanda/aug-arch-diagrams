# Authentication and Token Flow

This document describes the complete authentication chain from the browser through Orion to the running agent pod, including token acquisition, validation, and caching.

---

## Overview

The auth flow spans four trust boundaries:

```
Browser → Orion Backend → Red Hat Agent Operator → Agent Pod
   (session)    (Keycloak JWT)      (JWT audience)     (A2A request)
```

Each hop has its own authentication mechanism, and token management is handled centrally by the `KeycloakTokenManager`.

---

## 1. Browser to Orion Backend

The browser sends requests to the Orion backend with a **Backstage session cookie** (set during Backstage login). This cookie authenticates the user's session within the Backstage identity system.

---

## 2. Security Middleware

Every request to the augment backend routes passes through the security middleware. Behavior depends on `augment.security.mode` in app-config:

### Mode: `none`

- `requirePluginAccess`: **SKIPPED** — all requests pass without authentication
- Everyone is treated as admin
- Development only (see `07-security-modes.md`)

### Mode: `plugin-only`

- `requirePluginAccess`: Verifies valid Backstage session. Returns 401 if no valid session cookie.
- Admin check via `augment.security.adminUsers` list in app-config

### Mode: `full`

- `requirePluginAccess`: Checks `augment.access` permission via Backstage RBAC/permissions API
- Admin check via `augment.admin` permission through Backstage policy engine

---

## 3. User Identity Resolution

`getUserRef()` resolves the caller's identity:

```
httpAuth.credentials(req, { allow: ['user', 'none'] })
```

- For **authenticated users**: returns `principal.userEntityRef` (e.g., `user:default/alice`)
- For **unauthenticated requests** (when `none` is allowed): falls back to `user:default/guest`

This identity is used for:
- Ownership tracking (`createdBy` on agents)
- Permission checks (can this user edit/delete this agent?)
- Audit logging

---

## 4. Per-Request User Context

After resolving identity, the provider stores it for the request lifecycle:

```
provider.setUserContext(userRef)
```

This uses **AsyncLocalStorage** (Node.js async context tracking):
- Scoped to the current request only
- No race conditions between concurrent requests
- Available to any code in the request call chain without explicit parameter passing
- Used when the Operator needs `X-Backstage-User` header

---

## 5. Keycloak Token Acquisition

When the backend needs to communicate with the Red Hat Agent Operator, it acquires a service account token via `KeycloakTokenManager`.

### Standard Token: `getToken()`

1. Check in-memory cache (`cachedToken` + `expiresAt`)
2. If token is valid and not near expiry → return cached token immediately
3. If expired or missing → request new token from Keycloak:

```
POST <keycloak-base-url>/realms/<realm>/protocol/openid-connect/token

grant_type=client_credentials
client_id=kagenti-api
client_secret=<from-app-config>
```

4. Keycloak returns a JWT containing:
   - `aud`: audience claim with SPIFFE IDs for all registered agents (auto-managed by Operator controller)
   - `azp`: `kagenti-api` (authorized party)
   - `roles`: `[admin]` (service account role)
   - Standard claims: `exp`, `iat`, `iss`, `sub`

5. Token cached with lifetime = `(expires_in - 60s buffer)`. Minimum cache time: 10 seconds.

### Streaming Token: `getTokenForStreaming(minLifetimeMs)`

For streaming operations (SSE chat), tokens must remain valid for the duration of the stream:

1. Same cache check as `getToken()`
2. Additional check: remaining lifetime must be ≥ `minLifetimeMs` (default: 300,000ms = 5 minutes)
3. If current token has insufficient remaining lifetime → force refresh regardless of cache validity
4. This prevents token expiration mid-stream which would break the SSE connection

---

## 6. Request to Red Hat Agent Operator

Authenticated requests to the Operator include:

```
Authorization: Bearer <keycloak-jwt>
X-Backstage-User: <userRef>
Content-Type: application/json
```

- `Authorization`: The Keycloak service account JWT
- `X-Backstage-User`: The end-user's identity (for audit and ownership tracking)

---

## 7. Operator JWT Validation

The Red Hat Agent Operator validates the incoming JWT:

1. **Signature verification** — validates against Keycloak's public keys (JWKS endpoint)
2. **Expiry check** — rejects tokens past their `exp` claim
3. **Role verification** — confirms the `admin` role is present
4. **Issuer validation** — confirms token was issued by the expected Keycloak realm

---

## 8. Auth Bridge: Audience Validation

When requests reach the agent pod (for chat/A2A communication), the auth bridge sidecar performs additional validation:

1. Extracts the JWT from the Authorization header
2. Checks the `aud` (audience) claim
3. Verifies that the audience contains the **target agent's SPIFFE ID**
4. SPIFFE IDs follow the pattern: `spiffe://<trust-domain>/ns/<namespace>/sa/<agent-name>`

### On Success

Request is forwarded to the agent container via the A2A protocol on the loopback interface.

### On Audience Mismatch

- Returns **401 Unauthorized**
- The Orion backend catches this 401
- Clears the cached token (audience scope may have changed)
- Retries the request **once** with a freshly acquired token
- If retry fails: returns error to the frontend

---

## 9. Keycloak Audience Scope Management

Audience scopes are the mechanism that controls which agents a token can access:

### Auto-Creation

When a new agent is deployed, the Operator's Kubernetes controller:
1. Detects the new Deployment (via `kagenti.io/type=agent` label watch)
2. Creates a Keycloak **client scope** named `agent-{namespace}-{name}-aud`
3. The scope contains a mapper that adds the agent's SPIFFE ID to the token's `aud` claim
4. Adds this scope as a **default client scope** on the `kagenti-api` client

### Effect

- All tokens issued to `kagenti-api` automatically include audience claims for all registered agents
- When a new agent is added, existing cached tokens won't include its audience until they're refreshed
- This is why the retry-on-401 pattern exists: it handles the window between agent creation and token refresh

---

## 10. Token Lifecycle Summary

```
Token States:
┌─────────────────────────────────────────────────────┐
│ FRESH        │ Newly acquired, full lifetime        │
│ CACHED       │ In-memory, still valid               │
│ NEAR-EXPIRY  │ < 60s remaining, will be refreshed   │
│ EXPIRED      │ Past exp claim, must refresh          │
│ INVALIDATED  │ Cleared due to 401 (audience miss)   │
└─────────────────────────────────────────────────────┘
```

---

## 11. Security Properties

| Property | Mechanism |
|----------|-----------|
| No credential exposure to browser | Backend-only token, never sent to frontend |
| Per-agent access control | Audience claim + SPIFFE IDs |
| Token freshness | 60s buffer + streaming-aware refresh |
| Graceful rotation | Retry-on-401 handles scope updates |
| Request isolation | AsyncLocalStorage prevents user context leakage |
| Service identity | client_credentials grant (no user token forwarding) |
