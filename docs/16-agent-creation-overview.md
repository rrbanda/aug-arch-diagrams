# 16 - Agent Creation Overview

Complete map of all non-BYO agent creation methods in the Augment plugin.

## Entry Point

All creation flows route through `AgentCreateIntentDialog`, accessible from:
- **Marketplace** "My Agents" tab (+ New Agent button)
- **Command Center** KagentiAgentsPanel (New Agent button)
- **Agent Studio** Agent Catalog tab

The dialog presents two top-level choices: "Create Agent" (4 sub-methods) and "BYO Agent" (excluded from this diagram).

## The 5 Methods

### Method 1: Create using Skills
Compose an agent from reusable skills hosted in a Skills Marketplace. 3-step wizard: pick runtime, select skills, configure name/prompt. Deploys a DocsClaw pod to K8s. The only fully platform-managed creation path.

### Method 2: Create with Visual Canvas
Opens the Workflow Builder -- a no-code drag-and-drop editor. Define agent behavior using nodes (Agent, Tool, Classify, Logic, MCP, etc.) and edges (handoff, sequence, conditional). Published workflows become agents in the catalog.

### Method 3: Code Your Agent (DevSpaces)
Launch a cloud IDE workspace from framework-specific starter repos (Google ADK, LangGraph, CrewAI, OpenAI Agents, or custom). Development-only; the agent must be deployed later via BYO.

### Method 4: Create from Template
Browse Backstage Software Templates filtered by `spec.type: agent`. Run the Scaffolder to create a new Git repository with the agent project. Also development-only scaffolding.

### Method 5: Config-Driven (Admin Panel)
Use the AgentsPanel admin UI to define agents via form-based configuration. Supports multi-agent topologies with handoffs and tool delegation. Saved to AdminConfigService; runs on the Responses API orchestration layer.

## Governance Layer

All agent types converge into the `chatAgents` governance system with lifecycle states: draft -> pending -> published -> archived. Only published agents appear in the Marketplace for end users.
