# BYO Agent: Source Deploy Flow

This document describes the complete flow for deploying a BYO agent from a Git source repository through the Orion UI, using Shipwright to build the container image on-cluster.

---

## Prerequisites

Before using the Source from Git deployment method, the following must be in place:

| Requirement | Details |
|-------------|---------|
| **Shipwright installed** | Shipwright CRDs and controller running on the cluster. Without this, the "Source from Git" radio option is disabled in the wizard. |
| **Tekton Pipelines** | Shipwright uses Tekton underneath for TaskRun execution |
| **Build strategies created** | At minimum: `buildah` (for external registries with TLS) and `buildah-insecure-push` (for internal registries without valid TLS certs) |
| **Privileged SCC** | Build pods require privileged Security Context Constraints because buildah needs root access for container image building |
| **Push credentials** | Registry push credentials linked to the namespace's ServiceAccount (not specified in the wizard — see note below) |

---

## 1. Entry Point: New Agent Dialog

Same as Image Deploy: User clicks **"+ New Agent"** on My Agents tab → Intent Dialog → selects **BYO Agent** → opens `CreateAgentWizard`.

---

## 2. Step 1: Basics

Identical to Image Deploy:

- **Name**: DNS-1123 validated (lowercase alphanumeric + hyphens, max 63 chars)
- **Namespace**: from `GET /api/v1/namespaces?enabled_only=true` (labeled `kagenti-enabled=true`)
- **Protocol**: A2A (read-only)
- **Framework**: LangGraph default, or CrewAI, ADK, etc.

---

## 3. Step 2: Deployment

User selects the **"Source from Git"** radio option (only available when Shipwright is detected).

### Source Configuration

| Field | Required | Description |
|-------|----------|-------------|
| **Git URL** | Yes | HTTPS or SSH URL of the Git repository containing the agent source code |
| **Git Branch** | No | Branch to build from. Defaults to `main` |
| **Git Path** | No | Subdirectory within the repo containing the Dockerfile (for monorepos) |

### Registry Configuration

| Field | Required | Description |
|-------|----------|-------------|
| **Registry URL** | Yes | Target container registry where the built image will be pushed (e.g., `registry.example.com/org/my-agent`) |
| **Registry Secret** | No | Name of a Kubernetes Secret for registry push authentication. **Important note**: Leave this empty and instead link the secret to the namespace's ServiceAccount directly. The Red Hat Agent Operator does not map this field to Shipwright's `spec.output.credentials`, so specifying it here has no effect on the actual build. |

### Build Configuration

| Field | Description |
|-------|-------------|
| **Build Strategy** | Dropdown populated from `GET /api/v1/agents/build-strategies`. Options include `buildah`, `buildah-insecure-push`, and any custom strategies. |
| **Dockerfile Path** | Relative path to the Dockerfile within the Git path (default: `Dockerfile`) |
| **Build Timeout** | Maximum duration for the build before it is terminated |
| **Build Args** | Key-value pairs passed as `--build-arg` to the container build |

All of these fields are assembled into the `shipwrightConfig` field of the create request.

---

## 4. Step 3: Runtime

Identical to Image Deploy:

- Workload type (Deployment/StatefulSet/Job)
- Environment variables (direct value, Secret ref, ConfigMap ref)
- Service ports
- HTTP Route toggle
- Auth Bridge (proxy-sidecar/envoy-sidecar/lite/waypoint)
- SPIRE toggle
- mTLS mode

---

## 5. Submit: Trigger Build

When the user clicks **Submit**:

1. Frontend calls `buildRequest()` which maps `FormState` to `KagentiCreateAgentRequest` with `deploymentMethod=source`
2. Sends `POST /api/augment/kagenti/agents`

### Backend Processing

1. `validateNamespace()` — confirms namespace is in the allow-list
2. `KeycloakTokenManager.getToken()` — acquires service account JWT via client_credentials grant
3. Proxies full body to Red Hat Agent Operator: `POST /api/v1/agents` with `deploymentMethod=source`

### Operator Processing

1. Creates a **Shipwright Build** resource with the Git source, registry output, and build strategy
2. Triggers a **BuildRun** to start the build immediately

### Governance Auto-Registration

Backend automatically registers the `chatAgents` governance entry (identical to image deploy):
- `lifecycleStage: draft`
- `published: false`
- `visible: false`
- `createdBy: <userRef>`
- `source: kagenti`

---

## 6. Step 4: Build & Deploy Progress

After submit, the wizard advances to a **Step 4** progress view showing a timeline:

```
✓ Build submitted
◉ Building container image...
○ Deploying agent
○ Agent ready
```

### Polling Loop

The frontend polls the build status every **4 seconds**:

```
GET /api/augment/kagenti/agents/:namespace/:name/build-info
```

This returns the current `buildRunPhase` and related metadata from the Shipwright BuildRun resource.

---

## 7. Shipwright Build Execution

The Shipwright BuildRun orchestrates the following via Tekton:

1. **Clone** — Git repository cloned from the specified URL/branch/path
2. **Build** — Dockerfile executed via the buildah strategy (runs as privileged pod)
3. **Push** — Built container image pushed to the specified registry URL using credentials from the namespace ServiceAccount

Build phases reported:
- `Pending` — BuildRun created, waiting for pod scheduling
- `Running` — Build pod is executing
- `Succeeded` — Image built and pushed successfully
- `Failed` — Build encountered an error

---

## 8. Build Success: Finalize

When the polling detects `buildRunPhase=Succeeded`:

1. Frontend calls `POST /api/augment/kagenti/agents/:namespace/:name/finalize-build`
2. This signals the Operator to proceed with deployment
3. Operator creates Kubernetes resources from the newly-built image:
   - **Deployment** — runs the built container image
   - **Service** — ClusterIP routing to the agent pod
   - **Route** — external access (if HTTP Route was enabled)
4. Operator controller reconciles: creates Keycloak audience scope, registers agent card

Timeline updates:

```
✓ Build submitted
✓ Building container image
✓ Deploying agent
✓ Agent ready
```

Success banner is displayed. User clicks **"Done"** which triggers the `onCreated` callback and closes the wizard.

---

## 9. Build Failure Handling

If the build fails (`buildRunPhase=Failed`):

### Error Display

- Error message shown in the wizard with details from the BuildRun status
- Helpful kubectl command displayed for investigating logs:
  ```
  kubectl -n <namespace> logs -l build.shipwright.io/name=<build-name>
  ```

### Retry Build

- A **"Retry Build"** button is available
- Clicking it triggers a new BuildRun against the same Build resource
- The polling loop restarts from the beginning

---

## 10. Deploy Failure Handling

If the build succeeds but deployment fails:

- The timeline shows build as successful but deploy as failed
- A separate **"Retry Deploy"** button is available (distinct from build retry)
- This re-triggers only the deployment phase without rebuilding the image

---

## 11. Post-Creation

After successful build and deploy:

1. Wizard closes
2. My Agents tab refreshes
3. Agent appears with `source=kagenti`, `stage=draft`
4. Agent detail page available with Overview, Design, Test, Build, Agent Card tabs
5. Build tab shows build history (BuildRun records)
6. Test tab allows immediate interaction via InlineAgentChat

---

## Key Differences from Image Deploy

| Aspect | Image Deploy | Source Deploy |
|--------|--------------|---------------|
| Input | Container image URL | Git repository URL |
| Build step | None (image already exists) | Shipwright BuildRun (clone → build → push) |
| Wizard steps | 3 steps | 4 steps (includes Build & Deploy progress) |
| Time to deploy | Seconds (just creates K8s resources) | Minutes (build + push + deploy) |
| Prerequisites | None beyond Operator | Shipwright, Tekton, build strategies, privileged SCC |
| Retry granularity | Redeploy only | Retry build OR retry deploy independently |
