# Sprint Planning - Sprint Status Generator

<critical>The workflow execution engine is governed by: {project-root}/_bmad/core/tasks/workflow.xml</critical>
<critical>You MUST have already loaded and processed: {project-root}/_bmad/bmm/workflows/4-implementation/sprint-planning/workflow.yaml</critical>
<critical>🔧 AZURE DEVOPS INTEGRATION - Creates Feature work items for each epic when MCP is available</critical>

## 📚 Document Discovery - Full Epic Loading

**Strategy**: Sprint planning needs ALL epics and stories to build complete status tracking.

**Epic Discovery Process:**

1. **Search for whole document first** - Look for `epics.md`, `bmm-epics.md`, or any `*epic*.md` file
2. **Check for sharded version** - If whole document not found, look for `epics/index.md`
3. **If sharded version found**:
   - Read `index.md` to understand the document structure
   - Read ALL epic section files listed in the index (e.g., `epic-1.md`, `epic-2.md`, etc.)
   - Process all epics and their stories from the combined content
   - This ensures complete sprint status coverage
4. **Priority**: If both whole and sharded versions exist, use the whole document

**Fuzzy matching**: Be flexible with document names - users may use variations like `epics.md`, `bmm-epics.md`, `user-stories.md`, etc.

<workflow>

<!-- Azure DevOps MCP Preflight Check -->
<step n="0" goal="Check Azure DevOps MCP availability">
  <critical>AZURE DEVOPS MCP CONNECTION IS REQUIRED FOR SPRINT PLANNING</critical>
  <action>Try: Call MCP: list_work_items with wiql="SELECT [System.Id] FROM WorkItems WHERE [System.TeamProject] = '{azure_project_id}'" and top=1</action>
  <check if="MCP call succeeds">
    <action>Set azure_available = true</action>
    <output>ℹ️ Azure DevOps MCP connected - will create Feature work items for epics</output>
  </check>
  <check if="MCP call fails or times out">
    <output>🚫 Azure DevOps MCP not available</output>
    <output>Azure DevOps MCP connection is REQUIRED for this workflow.</output>
    <output>Please verify:</output>
    <output>   1. MCP server is running (check .mcp.json configuration)</output>
    <output>   2. PAT token is valid and not expired</output>
    <output>   3. Network connectivity to Azure DevOps is available</output>
    <output>   4. Org URL and project settings are correct</output>
    <action>HALT - Cannot proceed without Azure DevOps MCP connection</action>
  </check>
  <action>Continue to step 1</action>
</step>

<!-- Azure DevOps Iteration Creation -->
<step n="0.5" goal="Create or verify sprint iteration in Azure DevOps">
  <check if="azure_available == true AND auto_create_iterations == true">
    <critical>🔧 AZURE DEVOPS ITERATION CREATION - Creates sprint iteration before Features</critical>

    <!-- Sprint number detection: Azure first, then sprint-status.yaml fallback -->
    <action>Determine current sprint number:</action>
    <action>Query Azure DevOps for existing iterations to determine next sprint number</action>
    <action>Parse iteration names to find highest "Sprint N" pattern (e.g., "Sprint 1", "Sprint 2")</action>

    <check if="Azure returns iterations with Sprint pattern">
      <action>Extract highest sprint number from iteration names</action>
      <action>Set {{sprint_number}} = highest_sprint_number + 1</action>
      <output>ℹ️ Azure DevOps: Found existing sprints, starting Sprint {{sprint_number}}</output>
    </check>

    <check if="Azure returns no iterations or no Sprint pattern">
      <action>Check if {status_file} exists and contains current_sprint_number</action>
      <check if="status_file exists with current_sprint_number">
        <action>Set {{sprint_number}} = current_sprint_number from status_file</action>
      </check>
      <check if="status_file does NOT exist or missing current_sprint_number">
        <action>Set {{sprint_number}} = 1 (first sprint)</action>
      </check>
      <output>ℹ️ No existing sprints found, starting Sprint {{sprint_number}}</output>
    </check>

    <!-- Date calculation -->
    <action>Calculate sprint dates:</action>
    <action>Set {{start_date}} = sprint_start_date from config OR current date in ISO 8601 format</action>
    <action>Calculate {{finish_date}} = start_date + sprint_duration_days (format as ISO 8601)</action>
    <action>Example: If start_date = 2026-01-29 and sprint_duration_days = 14, then finish_date = 2026-02-12</action>

    <!-- Iteration creation -->
    <action>Create iteration in Azure DevOps:</action>
    <action>Call MCP: create_iteration with:
{
  "teamName": "{azure_team}",
  "name": "Sprint {{sprint_number}}",
  "startDate": "{{start_date}}",
  "finishDate": "{{finish_date}}",
  "description": "Sprint {{sprint_number}} for {project_name}"
}</action>

    <check if="iteration creation succeeds">
      <action>Set {{iteration_path}} = "{azure_project_id}\\{azure_team}\\Sprint {{sprint_number}}"</action>
      <output>✅ Azure DevOps: Created iteration "{{iteration_path}}" ({{start_date}} to {{finish_date}})</output>
    </check>

    <check if="iteration fails with error containing 'already exists' or 'conflict'">
      <action>Set {{iteration_path}} = "{azure_project_id}\\{azure_team}\\Sprint {{sprint_number}}"</action>
      <output>ℹ️ Azure DevOps: Iteration "{{iteration_path}}" already exists, reusing it</output>
    </check>

    <check if="iteration fails with other error">
      <output>⚠️ Azure DevOps: Could not create iteration (error: {{error_message}})</output>
      <output>Work items will be created without iteration path assignment</output>
      <action>Set {{iteration_path}} = null</action>
    </check>
  </check>

  <check if="azure_available == false OR auto_create_iterations == false">
    <action>Set {{iteration_path}} = null</action>
    <action>Set {{sprint_number}} = 1</action>
    <output>ℹ️ Skipping iteration creation (Azure unavailable or disabled in config)</output>
  </check>

  <action>Continue to Step 1</action>
</step>

<step n="1" goal="Parse epic files and extract all work items">
<action>Communicate in {communication_language} with {user_name}</action>
<action>Look for all files matching `{epics_pattern}` in {epics_location}</action>
<action>Could be a single `epics.md` file or multiple `epic-1.md`, `epic-2.md` files</action>

<action>For each epic file found, extract:</action>

- Epic numbers from headers like `## Epic 1:` or `## Epic 2:`
- Story IDs and titles from patterns like `### Story 1.1: User Authentication`
- Convert story format from `Epic.Story: Title` to kebab-case key: `epic-story-title`

**Story ID Conversion Rules:**

- Original: `### Story 1.1: User Authentication`
- Replace period with dash: `1-1`
- Convert title to kebab-case: `user-authentication`
- Final key: `1-1-user-authentication`

<action>Build complete inventory of all epics and stories from all epic files</action>
</step>

  <step n="0.5" goal="Discover and load project documents">
    <invoke-protocol name="discover_inputs" />
    <note>After discovery, these content variables are available: {epics_content} (all epics loaded - uses FULL_LOAD strategy)</note>
  </step>

<step n="2" goal="Build sprint status structure">
<action>For each epic found, create entries in this order:</action>

1. **Epic entry** - Key: `epic-{num}`, Default status: `backlog`
2. **Story entries** - Key: `{epic}-{story}-{title}`, Default status: `backlog`
3. **Retrospective entry** - Key: `epic-{num}-retrospective`, Default status: `optional`

**Example structure:**

```yaml
development_status:
  epic-1: backlog
  1-1-user-authentication: backlog
  1-2-account-management: backlog
  epic-1-retrospective: optional
```

</step>

<step n="3" goal="Apply intelligent status detection">
<action>For each story, detect current status by checking files:</action>

**Story file detection:**

- Check: `{story_location_absolute}/{story-key}.md` (e.g., `stories/1-1-user-authentication.md`)
- If exists → upgrade status to at least `ready-for-dev`

**Preservation rule:**

- If existing `{status_file}` exists and has more advanced status, preserve it
- Never downgrade status (e.g., don't change `done` to `ready-for-dev`)

**Status Flow Reference:**

- Epic: `backlog` → `in-progress` → `done`
- Story: `backlog` → `ready-for-dev` → `in-progress` → `review` → `done`
- Retrospective: `optional` ↔ `done`
  </step>

<step n="4" goal="Generate sprint status file">
<action>Create or update {status_file} with:</action>

**File Structure:**

```yaml
# generated: {date}
# project: {project_name}
# project_key: {project_key}
# tracking_system: {tracking_system}
# story_location: {story_location}

# STATUS DEFINITIONS:
# ==================
# Epic Status:
#   - backlog: Epic not yet started
#   - in-progress: Epic actively being worked on
#   - done: All stories in epic completed
#
# Epic Status Transitions:
#   - backlog → in-progress: Automatically when first story is created (via create-story)
#   - in-progress → done: Manually when all stories reach 'done' status
#
# Story Status:
#   - backlog: Story only exists in epic file
#   - ready-for-dev: Story file created in stories folder
#   - in-progress: Developer actively working on implementation
#   - review: Ready for code review (via Dev's code-review workflow)
#   - done: Story completed
#
# Retrospective Status:
#   - optional: Can be completed but not required
#   - done: Retrospective has been completed
#
# WORKFLOW NOTES:
# ===============
# - Epic transitions to 'in-progress' automatically when first story is created
# - Stories can be worked in parallel if team capacity allows
# - SM typically creates next story after previous one is 'done' to incorporate learnings
# - Dev moves story to 'review', then runs code-review (fresh context, different LLM recommended)

generated: { date }
project: { project_name }
project_key: { project_key }
tracking_system: { tracking_system }
story_location: { story_location }
current_sprint_number: {{sprint_number}}
current_iteration_path: {{iteration_path}}

development_status:
  # All epics, stories, and retrospectives in order
```

<action>Write the complete sprint status YAML to {status_file}</action>
<action>CRITICAL: Metadata appears TWICE - once as comments (#) for documentation, once as YAML key:value fields for parsing</action>
<action>Ensure all items are ordered: epic, its stories, its retrospective, next epic...</action>

<!-- Azure DevOps Feature creation for each epic -->
<check if="azure_available == true">
  <action>For each epic found in Step 1, create or find Azure Feature work item:</action>
  <action>Build create_work_item parameters:</action>
  <action>Set base_params = {
  "workItemType": "Feature",
  "title": "Epic {{epic_num}}: {{epic_title}}",
  "description": "<div><p>Epic {{epic_num}} from {project_name}</p></div>",
  "state": "New",
  "areaPath": "{azure_project_id}\\{azure_team}"
}</action>

  <check if="{{iteration_path}} is not null">
    <action>Add "iterationPath": "{{iteration_path}}" to base_params</action>
  </check>

  <action>Call MCP: create_work_item with: base_params</action>
  <action>Store returned Feature ID as {{azure_feature_id_{{epic_num}}}</action>
  <output>✅ Azure DevOps: Created Feature {{azure_feature_id_{{epic_num}}} for Epic {{epic_num}}</output>
  <output>   └─ Area: {azure_project_id}\\{azure_team}</output>
  <check if="{{iteration_path}} is not null">
    <output>   └─ Iteration: {{iteration_path}}</output>
  </check>
</check>
</step>

<step n="5" goal="Validate and report">
<action>Perform validation checks:</action>

- [ ] Every epic in epic files appears in {status_file}
- [ ] Every story in epic files appears in {status_file}
- [ ] Every epic has a corresponding retrospective entry
- [ ] No items in {status_file} that don't exist in epic files
- [ ] All status values are legal (match state machine definitions)
- [ ] File is valid YAML syntax

<action>Count totals:</action>

- Total epics: {{epic_count}}
- Total stories: {{story_count}}
- Epics in-progress: {{in_progress_count}}
- Stories done: {{done_count}}

<action>Display completion summary to {user_name} in {communication_language}:</action>

**Sprint Status Generated Successfully**

- **File Location:** {status_file}
- **Total Epics:** {{epic_count}}
- **Total Stories:** {{story_count}}
- **Epics In Progress:** {{epics_in_progress_count}}
- **Stories Completed:** {{done_count}}

**Next Steps:**

1. Review the generated {status_file}
2. Use this file to track development progress
3. Agents will update statuses as they work
4. Re-run this workflow to refresh auto-detected statuses

</step>

</workflow>

## Additional Documentation

### Status State Machine

**Epic Status Flow:**

```
backlog → in-progress → done
```

- **backlog**: Epic not yet started
- **in-progress**: Epic actively being worked on (stories being created/implemented)
- **done**: All stories in epic completed

**Story Status Flow:**

```
backlog → ready-for-dev → in-progress → review → done
```

- **backlog**: Story only exists in epic file
- **ready-for-dev**: Story file created (e.g., `stories/1-3-plant-naming.md`)
- **in-progress**: Developer actively working
- **review**: Ready for code review (via Dev's code-review workflow)
- **done**: Completed

**Retrospective Status:**

```
optional ↔ done
```

- **optional**: Ready to be conducted but not required
- **done**: Finished

### Guidelines

1. **Epic Activation**: Mark epic as `in-progress` when starting work on its first story
2. **Sequential Default**: Stories are typically worked in order, but parallel work is supported
3. **Parallel Work Supported**: Multiple stories can be `in-progress` if team capacity allows
4. **Review Before Done**: Stories should pass through `review` before `done`
5. **Learning Transfer**: SM typically creates next story after previous one is `done` to incorporate learnings
