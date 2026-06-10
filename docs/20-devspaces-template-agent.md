# 20 - DevSpaces and Template Agent Paths

These two paths scaffold development environments for building agents. They do not directly create running agents -- the user must later deploy via the BYO Agent path.

## Method 3: Code Your Agent (DevSpaces)

### Framework Picker

Frameworks are configured in `augment.devSpaces.frameworks`:

| Framework | Language | Starter Repo |
|-----------|----------|--------------|
| Google ADK | Python | Configurable per deployment |
| LangGraph | Python | LangChain-based graph agents |
| CrewAI | Python | Role-based multi-agent teams |
| OpenAI Agents | Python | Responses API agents |
| DIY / Custom | Any | User provides their own repo |

### Launch Flow

1. Pick framework -- pre-fills starter repo URL
2. Check DevSpaces health (`checkDevSpacesHealth()`)
3. Enter Git repo URL (optional CPU/memory config)
4. Create workspace (`POST /kagenti/devspaces`)
5. Poll until workspace is Running or Failed
6. Open cloud IDE with full development tooling

### After Development

User builds and pushes a container image, then returns to the platform and uses "BYO Agent" to deploy.

## Method 4: Create from Template (Backstage Scaffolder)

### Template Discovery

`AgentTemplateBrowser` queries the Backstage Catalog:
- Entity kind: `Template`
- Filter: `spec.type: agent`
- Falls back to all templates if none match

### Available Templates

| Template | Framework | What it scaffolds |
|----------|-----------|-------------------|
| Google ADK Agent | google-adk | Python ADK project with tool use and memory |
| LangGraph Agent | langgraph | Python LangGraph stateful agent |
| CrewAI Agent | crewai | Python CrewAI multi-agent team |

### Launch Flow

1. Browse template cards with descriptions and tags
2. "Use Template" opens Backstage Scaffolder in a new browser tab
3. Scaffolder collects parameters (agent name, owner, repo location)
4. Executes template steps (fetch skeleton, publish to GitHub, register in catalog)
5. Result: new Git repository with agent project scaffold

### DevSpace Integration

Templates can also be opened in DevSpaces directly -- the "Open in DevSpace" button passes the template's source repository URL to `DevSpacesLaunchForm`.

## Key Distinction

Neither path registers an agent in the platform's governance system. They produce code repositories or development workspaces. The agent lifecycle begins when the user deploys the built artifact through the BYO Agent wizard.
