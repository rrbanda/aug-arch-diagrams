# 10 - Tool Creation User Journey

Complete user journey for creating a new MCP tool, covering all three paths: Deploy, Develop from Template, and Develop via DevSpace.

## Entry Point

The user clicks **"+ Create Tool"** from the Marketplace or the Tools panel. This opens the `ToolCreateIntentDialog`.

## Path 1: Deploy (CreateToolWizard)

The most common path for deploying an existing MCP tool.

### Step 1: Basics
- **Name** - DNS-1123 compliant (lowercase alphanumeric + hyphens, max 63 chars)
- **Namespace** - Target Kubernetes namespace (dropdown if namespaces are available)
- **Description** - Brief description of the tool's purpose
- **Protocol** - `streamable_http` (recommended), `mcp`, `http`, or `grpc`
- **Framework** - `Python`, `Node.js`, or `Other` (sets a label, does not affect container)

### Step 2: Deployment Method
Two sub-paths:

**Image Deploy:**
- Container image URL (e.g., `quay.io/org/my-tool:latest`)
- Image pull secret (optional, for private registries)

**Source Deploy:**
- Git repository URL
- Git revision (branch/tag/commit)
- Context directory within the repo
- Container registry URL and secret for the built image
- Build strategy (e.g., Dockerfile)
- Dockerfile path, build arguments, build timeout

### Step 3: Runtime Configuration
- **Workload type** - `Deployment` or `StatefulSet`
- **Persistent storage** - Toggle + size (for StatefulSets)
- **Environment variables** - Direct value, from Secret, or from ConfigMap
- **Service ports** - Name, port, target port, protocol (TCP/UDP)
- **HTTP route** - Create an OpenShift Route for external access
- **Auth bridge** - Enable authentication sidecar
- **SPIRE** - Enable SPIFFE/SPIRE workload identity

### Submit
- **Image path:** Creates the tool immediately, shows success snackbar
- **Source path:** Enters Step 4 (Build & Deploy), polls Shipwright build status, then finalizes deployment

## Path 2: Develop from Template

1. Opens `AgentTemplateBrowser` filtered to `specType="tool"`
2. Discovers templates from the Backstage Catalog (`kind: Template`)
3. User selects a template and runs it via the Scaffolder
4. Creates a new Git repository with the MCP tool project scaffold
5. User can then open the repo in a DevSpace or later deploy via the wizard

## Path 3: Develop via Tool DevSpace

1. Opens `DevSpacesLaunchForm` with `resourceKind="tool"`
2. User provides a Git repository URL (can come from a template scaffold)
3. Launches a cloud IDE workspace with full tooling and runtime support
4. User develops the MCP tool in the cloud IDE
5. After development, user deploys via the wizard (Deploy path)

## Post-Creation

After the tool is created and running:
1. **Connect** - Discover MCP tools exposed by the server
2. **Invoke** - Test tools with arguments and view results
3. **Monitor** - Check route status, build info, deployment health
