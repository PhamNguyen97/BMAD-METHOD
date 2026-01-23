# Azure DevOps Integration Conventions

**Module**: bmad-azure
**Purpose**: Canonical conventions for integrating Azure DevOps work items with BMAD
**Version**: 1.0.0

---

## Overview

Azure DevOps is the **source of truth** for all task tracking, dependencies, and workflow state in BMAD. All workflows and agents interact with Azure DevOps through MCP tools, not through file-based tracking.

> **CRITICAL**: Azure DevOps state is authoritative. Markdown documents reflect Azure state but do not override it.

---

## Azure DevOps Configuration

The Azure DevOps MCP server is configured in `.mcp.json`:

```json
{
  "azureDevOps": {
    "orgUrl": "https://<azure-domain>/<azure-collection>",
    "project": "<azure-collection>",
    "teamBoard": "<azure-team>"
  }
}
```

---

## Work Item Hierarchy Mapping

| BMAD Concept | Azure DevOps Type | Parent Relationship | Example ID |
|--------------|------------------|---------------------|------------|
| **Epic** | Feature | None | `123456` |
| **Story** | User Story | Feature (parent) | `234567` |
| **Task** | Task | User Story (parent) | `345678` |
| **Subtask** | Task | Task (parent) | `456789` |
| **Review Finding** | Bug | User Story (parent) | `567890` |

### Link Type Reference Names

- **Parent-Child**: `System.LinkTypes.Hierarchy-Forward`
  - Use when: Creating parent-child relationships (Feature → User Story → Task)
  - Direction: Forward (parent to child)

- **Dependency**: `System.LinkTypes.Dependency-Forward`
  - Use when: Task B depends on Task A (blocking relationship)
  - Direction: Forward (dependent → blocked-by)

---

## State Mapping

### BMAD Stage → Azure State

| BMAD Stage | Azure DevOps State | Notes |
|------------|-------------------|-------|
| `backlog` | `New` | Initial state, not yet prioritized |
| `ready-for-dev` | `New` | Ready for development (add tag "Ready") |
| `in-progress` | `Active` | Work is actively being done |
| `review` | `Resolved` | Code complete, awaiting review |
| `done` | `Closed` | Work completed and verified |

### State Transition Flow

```
New → Active → Resolved → Closed
 ↑                    ↓
 └────────────────────┘
   (can reopen if needed)
```

---

## MCP Tool Usage Patterns

### 1. Query Work Items (WIQL)

**Pattern for finding ready User Stories:**
```xml
<action>Call MCP: list_work_items with:
{
  "wiql": "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType] FROM WorkItems WHERE [System.TeamProject] = '<azure-collection>' AND [System.WorkItemType] = 'User Story' AND [System.State] = 'Active' ORDER BY [System.Id]"
}
</action>
```

**Pattern for finding all Features:**
```xml
<action>Call MCP: list_work_items with:
{
  "wiql": "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.TeamProject] = '<azure-collection>' AND [System.WorkItemType] = 'Feature' ORDER BY [System.Id]"
}
</action>
```

### 2. Create Work Item

**Create a User Story under a Feature:**
```xml
<action>Call MCP: create_work_item with:
{
  "workItemType": "User Story",
  "title": "Story title here",
  "description": "<div><p>HTML formatted description</p></div>",
  "parentId": 123456,
  "priority": 2,
  "areaPath": "<azure-collection>\\<azure-team>"
}
</action>
```

**Create a Bug for code review findings:**
```xml
<action>Call MCP: create_work_item with:
{
  "workItemType": "Bug",
  "title": "Fix: Security vulnerability in authentication",
  "description": "<div><p>Found during code review...</p></div>",
  "parentId": 234567,
  "priority": 1,
  "areaPath": "<azure-collection>\\<azure-team>",
  "severity": "2 - High"
}
</action>
```

### 3. Update Work Item State

**Mark story as Active:**
```xml
<action>Call MCP: update_work_item with:
{
  "workItemId": 234567,
  "state": "Active"
}
</action>
```

**Mark task as Closed:**
```xml
<action>Call MCP: update_work_item with:
{
  "workItemId": 345678,
  "state": "Closed"
}
</action>
```

### 4. Create Parent-Child Link

**Link User Story to Feature:**
```xml
<action>Call MCP: manage_work_item_link with:
{
  "sourceWorkItemId": 234567,
  "targetWorkItemId": 123456,
  "operation": "add",
  "relationType": "System.LinkTypes.Hierarchy-Forward"
}
</action>
```

### 5. Create Dependency Link

**Task B depends on Task A:**
```xml
<action>Call MCP: manage_work_item_link with:
{
  "sourceWorkItemId": 456789,
  "targetWorkItemId": 345678,
  "operation": "add",
  "relationType": "System.LinkTypes.Dependency-Forward"
}
</action>
```

---

## Metadata Storage in Markdown Files

### YAML Frontmatter Pattern

```yaml
---
title: "User Authentication Story"
azure_feature_id: 123456
azure_story_id: 234567
azure_project: <azure-collection>
azure_org_url: https://<azure-domain>/<azure-collection>
---
```

### Alternative: Dedicated Section

```markdown
## Azure DevOps Tracking

- **Feature ID**: [123456](https://<azure-domain>/<azure-collection>/_workitems/edit/123456)
- **User Story ID**: [234567](https://<azure-domain>/<azure-collection>/_workitems/edit/234567)
- **State**: Active
- **Last Sync**: 2026-01-22T10:30:00Z
```

---

## Title and Description Conventions

### Title Patterns

- **Features**: `Epic N: Descriptive name` (e.g., `Epic 1: User Authentication`)
- **User Stories**: `Story N-M: Actionable title` (e.g., `Story 1-2: Login form validation`)
- **Tasks**: Start with verb (e.g., `Implement login form`, `Add unit tests for auth`)
- **Bugs**: `Fix: Brief description` (e.g., `Fix: Null pointer in auth service`)

### Description Format (HTML)

```html
<div>
  <h3>Overview</h3>
  <p>Brief description of what this work item encompasses.</p>

  <h3>Acceptance Criteria</h3>
  <ul>
    <li>Criterion 1</li>
    <li>Criterion 2</li>
  </ul>

  <h3>Technical Notes</h3>
  <p>Any technical implementation details.</p>
</div>
```

---

## Priority Mapping

| BMAD Priority | Azure DevOps Priority | Use Case |
|---------------|----------------------|----------|
| Critical | 1 | Review findings (high severity), Blockers |
| High | 2 | Stories, Tasks, Features |
| Normal | 3 | Subtasks, Low-severity findings |
| Low | 4 | Nice-to-have items |

---

## Area and Iteration Path Conventions

### Area Path Format
```
<azure-collection>\<azure-team>\[Optional Feature Area]
```

### Iteration Path Format

The iteration path of a team should be inside the team area:

```
<azure-collection>\<azure-team>\Sprint N\[Optional Iteration]
```

**Examples:**
- `MyProject\MyTeam\Sprint 1`
- `MyProject\MyTeam\Sprint 2\Iteration 1`
- `MyProject\MyTeam\Sprint 3\Iteration 2`

---

## Error Handling

### MCP Connection is Required

**CRITICAL**: Azure DevOps MCP connection is **REQUIRED** for all BMAD workflows. Workflows will HALT if MCP is unavailable.

```xml
<check if="MCP call fails with 'connection refused' or 'timeout'">
  <output>🚫 Azure DevOps MCP not available</output>
  <output>Azure DevOps MCP connection is REQUIRED for this workflow.</output>
  <output>Please verify:</output>
  <output>   1. MCP server is running (check .mcp.json configuration)</output>
  <output>   2. PAT token is valid and not expired</output>
  <output>   3. Network connectivity to Azure DevOps is available</output>
  <output>   4. Org URL and project settings are correct</output>
  <action>HALT - Cannot proceed without Azure DevOps MCP connection</action>
</check>
```

### When Work Item Not Found

```xml
<check if="MCP returns 'Work item not found'">
  <output>WARNING: Work item {id} not found in Azure DevOps</output>
  <output>It may have been deleted or the ID is incorrect</output>
  <ask>Options:
  1) Create new work item in Azure
  2) Update the ID in the markdown file
  3) Skip and continue</ask>
</check>
```

---

## Sync Agent Commands

The azure-sync agent supports these commands:

### full-sync
Complete bidirectional synchronization:
1. Parse MD files for tasks/stories
2. Create/update Azure work items
3. Pull Azure state back to MD files
4. Report any discrepancies

### md-to-azure
One-way sync from markdown to Azure:
1. Parse MD checklists
2. Create new Azure work items
3. Update Azure work item titles/descriptions
4. Store returned IDs in MD metadata

### azure-to-md
One-way sync from Azure to markdown:
1. Query Azure for all BMAD-related work items
2. Update MD status sections
3. Update MD metadata with latest state
4. Report changes

### migrate-stories
Bulk migration workflow:
1. Scan all story files
2. Create corresponding Azure work items
3. Establish parent-child relationships
4. Set up dependencies
5. Verify migration success

---

## Common Workflow Patterns

### Starting Work on a Story

1. Query Azure for "New" User Stories tagged "Ready"
2. Select first available story
3. Call `update_work_item` to set state to "Active"
4. Begin implementation

### Completing a Task

1. Implement and test the task
2. Call `update_work_item` to set state to "Closed"
3. Check if all tasks under the story are closed
4. If yes, set story state to "Resolved"

### Creating Review Findings

1. For each high/medium severity finding:
   - Create Bug work item
   - Set parent to the User Story
   - Create dependency link (story depends on bug fix)
2. For low severity findings:
   - Create Task work item (optional)

### Sprint Status Report

1. Query Azure for all User Stories in the project
2. Count by state: New, Active, Resolved, Closed
3. Map to BMAD stages for reporting
4. Display next recommended action

---

## Migration Tool Usage

### migrate-bmad-to-azure.js

```bash
# Dry run (safe preview)
node tools/azure/migrate-bmad-to-azure.js --dry-run --verbose

# Actual migration
node tools/azure/migrate-bmad-to-azure.js --verbose
```

**What it does:**
- Parses epic files (`epics.md` or `epics/` directory)
- Parses story files for tasks/subtasks
- Creates Features for each epic
- Creates User Stories for each story
- Creates Tasks for each task/subtask
- Sets up sequential blocking dependencies
- Preserves existing Azure IDs (for incremental migration)
- Marks completed items as "Closed"

### reconcile-bmad-azure.js

```bash
# Check for discrepancies
node tools/azure/reconcile-bmad-azure.js --verbose

# Auto-fix discrepancies
node tools/azure/reconcile-bmad-azure.js --fix --verbose
```

**Checks performed:**
- Status mismatch: sprint-status.yaml vs Azure state
- Story status: story file Status vs Azure state
- Task completion: checkboxes vs Azure "Closed" state
- Missing work items: IDs in docs that don't exist in Azure
- Orphaned work items: Azure items without MD references

**Resolution**: Azure is authoritative (docs updated to match Azure)

---

## Verification Checklist

After implementing Azure DevOps integration, verify:

- [ ] MCP server is configured in `.mcp.json`
- [ ] `azure-conventions.md` is installed to `_bmad/bmm/data/`
- [ ] `bmad-azure` module appears in module selection
- [ ] `list_work_items` MCP call returns work items
- [ ] `create_work_item` creates items visible in Azure portal
- [ ] `update_work_item` changes are visible in Azure portal
- [ ] `manage_work_item_link` creates dependencies visible in Azure portal
- [ ] sprint-status workflow queries Azure for Active User Stories
- [ ] dev-story workflow creates/updates Azure work items
- [ ] create-story workflow creates User Stories with parent Features
- [ ] code-review workflow creates Bugs with dependency links
- [ ] azure-sync agent performs bidirectional synchronization

---

## Troubleshooting

### MCP Connection Issues

**Symptom**: MCP calls fail with "connection refused"

**Solution**:
1. Verify `.mcp.json` configuration is correct
2. Check that MCP server path is valid
3. Verify PAT token has not expired
4. Test MCP server manually with a simple query

### Work Item Not Found

**Symptom**: `get_work_item` returns "Work item not found"

**Solution**:
1. Verify the work item ID is correct
2. Check that the work item exists in Azure portal
3. Verify you have access to the project/team
4. Check that the work item hasn't been deleted

### Permission Denied

**Symptom**: MCP calls fail with "403 Forbidden"

**Solution**:
1. Verify PAT token has "Work Items" (Read & Write) scope
2. Check that you're a member of the allowed team board
3. Verify area path permissions

### State Transition Not Allowed

**Symptom**: `update_work_item` fails with "state transition not allowed"

**Solution**:
1. Check Azure DevOps process template (Agile, Scrum, CMMI)
2. Verify the state transition is valid for your template
3. Some templates require different state names

---

## References

- Azure DevOps REST API: https://learn.microsoft.com/en-us/rest/api/azure/devops/
- WIQL Syntax: https://learn.microsoft.com/en-us/azure/devops/boards/queries/wiql-syntax
- Work Item Types: https://learn.microsoft.com/en-us/azure/devops/boards/work-items/work-item-type
