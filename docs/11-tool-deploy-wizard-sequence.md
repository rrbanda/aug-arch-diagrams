# 11 - Tool Deploy Wizard Sequence

Step-by-step sequence of the `CreateToolWizard` for deploying MCP tools, covering both image and source deployment paths.

## Key Components

| Component | File | Role |
|-----------|------|------|
| `CreateToolWizard` | `CreateToolWizard.tsx` | Wizard orchestrator |
| `useToolWizardForm` | `useToolWizardForm.ts` | All form state, validation, submit, build polling |
| `WizardShell` | `WizardShell.tsx` | Shared MUI Dialog + Stepper |
| `ToolWizardBasicsStep` | `ToolWizardBasicsStep.tsx` | Step 1: identity fields |
| `ToolWizardDeployStep` | `ToolWizardDeployStep.tsx` | Step 2: image vs source |
| `ToolWizardRuntimeStep` | `ToolWizardRuntimeStep.tsx` | Step 3: workload config |
| `AgentWizardBuildStep` | `AgentWizardBuildStep.tsx` | Step 4: build polling (source only) |
| `toolWizardUtils` | `toolWizardUtils.ts` | `buildToolRequest()` form-to-API mapper |

## API Request Body

The wizard builds a `KagentiCreateToolRequest`:

```
{
  name: string,           // DNS-1123 name
  namespace: string,      // K8s namespace
  protocol?: string,      // streamable_http, mcp, http, grpc
  framework?: string,     // Python, nodejs
  description?: string,
  deploymentMethod?: 'image' | 'source',
  containerImage?: string,
  imagePullSecret?: string,
  gitUrl?: string,
  gitRevision?: string,
  contextDir?: string,
  registryUrl?: string,
  registrySecret?: string,
  imageTag?: string,
  shipwrightConfig?: { buildStrategy, dockerfilePath, buildArgs, buildTimeout },
  workloadType?: 'deployment' | 'statefulset',
  persistentStorage?: { enabled, size },
  envVars?: [{ name, value?, source, refName?, refKey? }],
  servicePorts?: [{ name, port, targetPort, protocol }],
  createHttpRoute?: boolean,
  authBridgeEnabled?: boolean,
  spireEnabled?: boolean,
}
```

## Image Deploy Sequence

1. User fills Steps 1-3, clicks "Create"
2. `buildToolRequest(formState)` constructs the API body
3. `POST /kagenti/tools` with `deploymentMethod: 'image'`
4. Kagenti creates K8s Deployment + Service immediately
5. Wizard closes, shows success snackbar, refreshes tool list

## Source Deploy Sequence

1. User fills Steps 1-3 with Git source config, clicks "Build & Deploy"
2. `POST /kagenti/tools` with `deploymentMethod: 'source'`
3. Kagenti creates Shipwright Build CRD
4. Wizard advances to Step 4 (Build & Deploy)
5. Polls `GET /kagenti/tools/:ns/:name/build-info` every 5 seconds
6. Displays build phase, start time, logs link
7. On build completion, calls `POST /kagenti/tools/:ns/:name/finalize-build`
8. Finalize creates the K8s Deployment from the built image
9. Wizard closes, shows success
