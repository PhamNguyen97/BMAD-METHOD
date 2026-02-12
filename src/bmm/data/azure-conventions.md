# Azure DevOps Integration Conventions

**Module**: bmm
**Purpose**: Canonical conventions for integrating Azure DevOps work items with BMAD
**Version**: 6.0.0

---

## Overview

Azure DevOps is the **source of truth** for all task tracking, dependencies, and workflow state in BMAD. All workflows and agents interact with Azure DevOps through MCP tools, not through file-based tracking.

> **CRITICAL**: Azure DevOps state is authoritative. Markdown documents reflect Azure state but do not override it.

---

## Azure DevOps Configuration

The Azure DevOps MCP server is configured in `.mcp.json`:

```json
{
  "azure-devops":{
    "name": "azure-devops",
    "url": "http://azure-mcp.url/mcp",
    "type": "http",
    "headers": {
      "X-AzureDevOps-Org": "https://<azure-domain>/{azure_collection}",       
      "X-AzureDevOps-PAT": "your pat",
      "X-AzureDevOps-Default-Project": "{azure_project}",
      "X-AzureDevOps-Allowed-Team-Boards": "{azure_team}"
    }
  }
}
```

---

## Work Item Hierarchy Mapping

| BMAD Concept | Azure DevOps Type | Parent Relationship | Example ID |
|--------------|------------------|---------------------|------------|
| **Epic** | Epic | None | `123456` |
| **Story** | User Story | Epic (parent) | `234567` |
| **Task** | Task | User Story (parent) | `345678` |
| **Subtask** | Task | Task (parent) | `456789` |
| **Review Finding** | Bug | User Story (parent) | `567890` |

### Link Type Reference Names

- **Parent-Child**: `System.LinkTypes.Hierarchy-Forward`
  - Use when: Creating parent-child relationships (Epic → User Story → Task)
  - Direction: Forward (parent to child)

- **Dependency (Predecessor → Successor)**: `System.LinkTypes.Dependency-Forward`
  - **Predecessor**: Task that must complete FIRST (blocks other tasks)
  - **Successor**: Task that depends on the predecessor (blocked by predecessor)
  - Use when: Task B (Successor) depends on Task A (Predecessor) - A must complete before B can start
  - Direction: Forward (Successor → Predecessor) - the dependent task links to the task it depends on
  - **Example**: If Task A is Predecessor of Task B, then B is Successor of A (A then B)

### Predecessor and Successor Relationship Summary

| Relationship | Meaning | Order | Azure Link Direction |
|--------------|---------|-------|---------------------|
| **A is Successor of B** | B must complete before A can start | B → A | A links to B |
| **A is Predecessor of B** | A must complete before B can start | A → B | B links to A |

**Key Points:**
- The **Successor** (dependent task) always creates the link to the **Predecessor** (blocking task)
- If Task A is Predecessor of Task B: Task B will link to Task A
- If Task A is Successor of Task B: Task A will link to Task B
- In both cases: Predecessor completes FIRST, then Successor can start

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
  "wiql": "SELECT [System.Id], [System.Title], [System.State], [System.WorkItemType] FROM WorkItems WHERE [System.TeamProject] = '{azure_collection}' AND [System.WorkItemType] = 'User Story' AND [System.State] = 'Active' ORDER BY [System.Id]"
}
</action>
```

**Pattern for finding all Epics:**
```xml
<action>Call MCP: list_work_items with:
{
  "wiql": "SELECT [System.Id], [System.Title], [System.State] FROM WorkItems WHERE [System.TeamProject] = '{azure_collection}' AND [System.WorkItemType] = 'Epic' ORDER BY [System.Id]"
}
</action>
```

### 2. Create Work Item

**Create a User Story under an Epic (include Target Date and Acceptance Criteria):**
```xml
<action>Call MCP: create_work_item with:
{
  "workItemType": "User Story",
  "title": "Story title here",
  "description": "<div><p>HTML formatted description</p></div>",
  "acceptanceCriteria": "<div><ul><li>Criterion 1</li><li>Criterion 2</li></ul></div>",
  "parentId": 123456,
  "priority": 2,
  "areaPath": "{azure_collection}\\{azure_team}",
  "additionalFields": {
    "Microsoft.VSTS.Scheduling.TargetDate": "2026-01-30"
  }
}
</action>
```

**Note:** Use separate `description` and `acceptanceCriteria` fields. Do NOT embed acceptance criteria in the description.

**Create a Bug for code review findings:**
```xml
<action>Call MCP: create_work_item with:
{
  "workItemType": "Bug",
  "title": "Fix: Security vulnerability in authentication",
  "description": "<div><p>Found during code review...</p></div>",
  "parentId": 234567,
  "priority": 1,
  "areaPath": "{azure_collection}\\{azure_team}",
  "severity": "2 - High"
}
</action>
```

### 3. Update Work Item State

**CRITICAL: Always include date fields when updating work item time tracking**

When updating work items in the tracking system, always add "Start Date" and "Target Date" (not "End Date"):

- **Start Date** (`Microsoft.VSTS.Scheduling.StartDate`): Set when work begins (state → Active)
- **Target Date** (`Microsoft.VSTS.Scheduling.TargetDate`): Set when work is ready for review/completion (state → Resolved/Closed)

**Mark story as Active (include Start Date):**
```xml
<action>Call MCP: update_work_item with:
{
  "workItemId": 234567,
  "state": "Active",
  "additionalFields": {
    "Microsoft.VSTS.Scheduling.StartDate": "2026-01-23"
  }
}
</action>
```

**Mark story as Resolved (include Target Date):**
```xml
<action>Call MCP: update_work_item with:
{
  "workItemId": 234567,
  "state": "Resolved",
  "additionalFields": {
    "Microsoft.VSTS.Scheduling.TargetDate": "2026-01-25"
  }
}
</action>
```

**Mark task as Closed (include Target Date):**
```xml
<action>Call MCP: update_work_item with:
{
  "workItemId": 345678,
  "state": "Closed",
  "additionalFields": {
    "Microsoft.VSTS.Scheduling.TargetDate": "2026-01-24"
  }
}
</action>
```

### 4. Create Parent-Child Link

**Link User Story to Epic:**
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

### 5. Create Dependency Link (Predecessor → Successor)

**Task B (Successor) depends on Task A (Predecessor):**
- Task A (Predecessor) must complete FIRST
- Task B (Successor) is blocked by Task A
- B is Successor of A (A then B)
- A is Predecessor of B (A then B)

```xml
<action>Call MCP: manage_work_item_link with:
{
  "sourceWorkItemId": 456789,  <!-- Task B (Successor - dependent task) -->
  "targetWorkItemId": 345678,  <!-- Task A (Predecessor - must complete first) -->
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
azure_epic_id: 123456
azure_story_id: 234567
azure_project: {azure_collection}
azure_org_url: https://<azure-domain>/{azure_collection}
---
```

### Alternative: Dedicated Section

```markdown
## Azure DevOps Tracking

- **Epic ID**: [123456](https://<azure-domain>/{azure_collection}/_workitems/edit/123456)
- **User Story ID**: [234567](https://<azure-domain>/{azure_collection}/_workitems/edit/234567)
- **State**: Active
- **Last Sync**: 2026-01-22T10:30:00Z
```

---

## Title and Description Conventions

### Title Patterns

- **Epics**: `Epic N: Descriptive name` (e.g., `Epic 1: User Authentication`)
- **User Stories**: `Story N-M: Actionable title` (e.g., `Story 1-2: Login form validation`)
- **Tasks**: Start with verb (e.g., `Implement login form`, `Add unit tests for auth`)
- **Bugs**: `Fix: Brief description` (e.g., `Fix: Null pointer in auth service`)

### Field Format (Separate Fields)

**Use separate `description` and `acceptanceCriteria` fields when creating work items.**

**Description field** - HTML format with overview and technical details:
```html
<div>
  <h3>Overview</h3>
  <p>Brief description of what this work item encompasses.</p>

  <h3>Technical Notes</h3>
  <p>Any technical implementation details.</p>
</div>
```

**Acceptance Criteria field** - HTML format with bullet points (NO heading):
```html
<div>
  <ul>
    <li>Criterion 1</li>
    <li>Criterion 2</li>
  </ul>
</div>
```

**Note:** Do NOT include "Acceptance Criteria" or "##" heading - only the list items.

---

## Priority Mapping

| BMAD Priority | Azure DevOps Priority | Use Case |
|---------------|----------------------|----------|
| Critical | 1 | Review findings (high severity), Blockers |
| High | 2 | Stories, Tasks, Epics |
| Normal | 3 | Subtasks, Low-severity findings |
| Low | 4 | Nice-to-have items |

---

## Area and Iteration Path Conventions

### Area Path Format
```
{azure_collection}\{azure_team}\[Optional Feature Area]
```

### Iteration Path Format

The iteration path of a team should be inside the team area:

```
{azure_collection}\{azure_team}\Sprint N\[Optional Iteration]
```

**Examples:**
- `MyProject\MyTeam\Sprint 1`
- `MyProject\MyTeam\Sprint 2\Iteration 1`
- `MyProject\MyTeam\Sprint 3\Iteration 2`

### Iteration Creation in Sprint Planning

**When creating sprint iterations:**

1. **Determine sprint number**: Auto-detect from Azure DevOps (query existing iterations) OR use `current_sprint_number` from config as fallback
2. **Calculate dates**:
   - `startDate`: Use `sprint_start_date` from config or current date (ISO 8601 format)
   - `finishDate`: `startDate + sprint_duration_days` (default: 14 days)
3. **Create iteration**: Use MCP `create_iteration` before creating Features
4. **Assign iterationPath**: Format `{azure_collection}\{azure_team}\Sprint {{sprint_number}}`

**Iteration Path Assignment Pattern:**

```xml
<action>Call MCP: create_work_item with:
{
  "workItemType": "Epic",
  "title": "Epic {{epic_num}}: {{epic_title}}",
  "iterationPath": "{azure_collection}\\{azure_team}\\Sprint {{sprint_number}}",
  "areaPath": "{azure_collection}\\{azure_team}",
  "state": "New"
}</action>
```

**Error Handling:**
- If iteration already exists: Reuse existing iteration, log info message
- If iteration creation fails: Log warning, continue without iteration path assignment
- If `auto_create_iterations` is false: Skip iteration creation entirely

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
3. Establish parent-child relationships (Epic → User Story → Task)
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
   - Create dependency link: Story (Successor) depends on Bug (Predecessor)
     - Bug is Predecessor (must fix first)
     - Story is Successor (blocked until bug is fixed)
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
- Creates Epics for each epic
- Creates User Stories for each story
- Creates Tasks for each task/subtask
- Sets up sequential blocking dependencies
- Preserves existing Azure IDs (for incremental migration)
- Marks completed items as "Closed"

### reconcile-azure.js

```bash
# Check for discrepancies
node tools/azure/reconcile-azure.js --verbose

# Auto-fix discrepancies
node tools/azure/reconcile-azure.js --fix --verbose
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
- [ ] `bmm` module is installed with Azure configuration
- [ ] `list_work_items` MCP call returns work items
- [ ] `create_work_item` creates items visible in Azure portal
- [ ] `update_work_item` changes are visible in Azure portal
- [ ] `manage_work_item_link` creates dependencies visible in Azure portal
- [ ] sprint-status workflow queries Azure for Active User Stories
- [ ] dev-story workflow creates/updates Azure work items
- [ ] create-story workflow creates User Stories with parent Epics
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

- Azure DevOps REST API: <https://learn.microsoft.com/en-us/rest/api/azure/devops/>
- WIQL Syntax: <https://learn.microsoft.com/en-us/azure/devops/boards/queries/wiql-syntax>
- Work Item Types: <https://learn.microsoft.com/en-us/azure/devops/boards/work-items/work-item-type>
