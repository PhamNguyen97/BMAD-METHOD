# BMAD-Azure: Azure DevOps Integration Module

Integrates Azure DevOps as the source of truth for task tracking, dependencies, and workflows across all BMAD agents.

## Overview

This module transforms BMAD from file-based task tracking to Azure DevOps integrated work item tracking. It uses the Azure DevOps MCP server (configured in `.mcp.json`) to:

- Create and manage Features, User Stories, Tasks, and Bugs
- Track work item states (New → Active → Resolved → Closed)
- Establish parent-child and dependency relationships
- Synchronize state between Azure DevOps and markdown documents

## Work Item Hierarchy

```
Feature (Epic)
├── User Story (Story)
│   ├── Task
│   │   └── Subtask (Task)
│   └── Review Finding (Bug)
```

## State Mapping

| BMAD Stage | Azure DevOps State |
|------------|-------------------|
| backlog | New |
| ready-for-dev | New (+ tag "Ready") |
| in-progress | Active |
| review | Resolved |
| done | Closed |

## Components

- **azure-conventions.md**: Canonical conventions for MCP tool usage and work item mapping
- **azure-sync.agent.yaml**: Bidirectional sync agent for MD ↔ Azure synchronization
- **Migration tools**: Scripts to migrate existing BMAD projects to Azure DevOps

## Setup

1. Ensure Azure DevOps MCP server is configured in `.mcp.json`
2. Install BMAD with the `bmad-azure` module (default selected)
3. Run migration tools if migrating existing projects

## Usage

See [azure-conventions.md](../../bmm/data/azure-conventions.md) for detailed usage patterns.
