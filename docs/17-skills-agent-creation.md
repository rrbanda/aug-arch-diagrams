# 17 - Skills-Based Agent Creation

The skills-based path is the only fully platform-managed agent creation method that deploys a running agent without requiring a user-supplied container image.

## Flow Summary

1. **Select Runtime** -- choose from configured runtimes (e.g., DocsClaw in Go, ZeroClaw in Rust)
2. **Choose Skills** -- browse and select skills from an external Skills Marketplace API
3. **Configure Agent** -- set name, namespace, and system prompt (auto-derived from selected skills)
4. **Deploy** -- platform generates K8s manifests and deploys the agent pod

## Runtime Selection

Runtimes are configured in `augment.skillsMarketplace.runtimes` in app-config.yaml. Each runtime specifies:
- Container image for the agent pod
- Init image for skill fetching
- Language, footprint, and features

## Skill Selection

Skills are fetched from an external marketplace API (`augment.skillsMarketplace.baseUrl`). Each skill has:
- Name, description, domain (for grouping)
- Plugin name, Git path, slug

## K8s Deployment

The backend generates and applies:
- **Namespace** (auto-derived: `{username}-agents`)
- **Secret** (LLM API keys, marketplace credentials)
- **ConfigMap** (skill definitions as YAML)
- **Service** (ClusterIP for the agent pod)
- **Deployment** (init container fetches skills, runtime container serves MCP/A2A)

## Governance Registration

On successful deploy, a `chatAgents` entry is created with:
- `lifecycleStage: 'draft'`
- `framework: 'docsclaw'`
- `source: 'skills'`
- `chatEndpoint: http://{service}:{port}`

## Health Polling

The UI polls `GET /skills/agents/{name}/info` every 5 seconds (up to 12 attempts) to check pod health before showing the success screen.
