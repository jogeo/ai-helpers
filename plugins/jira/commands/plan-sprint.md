---
description: Plan a sprint with intelligent work prioritization based on epics and backlog
argument-hint: "<project-key> [component] [--max-points number]"
---

## Name
jira:plan-sprint

## Synopsis
```
/jira:plan-sprint <project-key> [component] [--max-points number]
```

## Description
The `jira:plan-sprint` command helps teams plan upcoming sprints by identifying critical work items and intelligently selecting backlog items based on priority, rank, and capacity constraints.

This command is particularly useful for:
- Sprint planning meetings
- Prioritizing work linked to active epics in development
- Including OCPBUGS bugs in ON_QA status that need verification
- Identifying routine quality work (maintenance releases, RCA, daily regression monitoring)
- Capacity-based sprint scoping with story point limits
- Identifying high-priority backlog items

**Component Filtering (Optional)**:
- When a component is specified (e.g., "QE"), the command filters epic-linked tasks and backlog items to only include issues with that component
- When no component is specified, the command includes ALL issues across all components in the project
- Multi-component support: Specifying a component (e.g., "QE") will match all issues that include that component, regardless of whether they have additional components assigned

## Key Features

- **OCPBUGS ON_QA Integration** – Automatically identifies bugs in ON_QA status that need verification
  - Prompts user for OCPBUGS component(s) to include
  - Searches for bugs in ON_QA status matching the specified components
  - Includes these bugs as verification work in the sprint plan

- **Routine Quality Work Detection** – Identifies ongoing quality responsibilities
  - Maintenance releases (z-stream, errata leadership)
  - Root Cause Analysis ownership for escaped bugs
  - Daily regression monitoring and test triage responsibilities
  - Warns if routine work items are missing and suggests creating them

- **Active Epic Prioritization** – Prioritizes work linked to active epics through intelligent epic state analysis:
  - Epics in "Dev Complete" state
  - Epics in "In Progress" state
  - Epics in "Testing" state
  - Epics with linked stories in "Review" state

- **Intelligent Backlog Selection** – Selects additional work based on:
  - Priority (Blocker, Critical, Major) - highest priority first
  - Rank within each priority tier - respects backlog ordering as ranked by the team
  - Story point capacity limits (optional)
  - **Excludes items linked to inactive epics** (NEW, PLANNING, TO DO states) - only selects standalone items or items linked to active epics

- **Capacity Management** – Respects story point limits and provides clear capacity utilization reporting

## Implementation

The `jira:plan-sprint` command runs in six main phases:

### 💬 Phase 1: User Input - OCPBUGS Component(s)

**Prompt the user to provide OCPBUGS component information.**

1. Use the `AskUserQuestion` tool to ask the user which OCPBUGS component(s) should be included for ON_QA bug verification:
   - Question: "Which OCPBUGS component(s) should be included for bug verification in this sprint?"
   - Provide hint: "Provide one or more OCPBUGS component names (e.g., 'Microshift', 'Logical Volume Manager Storage'). Leave empty to skip OCPBUGS bug inclusion."
   - Allow multi-line or comma-separated input

2. **If user provides component(s):**
   - Parse the component names (handle comma-separated or multi-line input)
   - Store component list for use in Phase 2
   - Proceed with OCPBUGS ON_QA search in Phase 2

3. **If user skips or provides empty input:**
   - Skip Phase 2 entirely
   - Add informational note in final report that OCPBUGS bugs were not included
   - Proceed directly to Phase 3

### 🐛 Phase 2: OCPBUGS ON_QA Identification

**This phase only runs if the user provided OCPBUGS component(s) in Phase 1.**

1. Search for bugs in ON_QA status with the specified component(s):
   ```jql
   project = OCPBUGS AND
   status = ON_QA AND
   component IN ({component-list})
   ORDER BY priority DESC, updated DESC
   ```

   **Important**: Include relevant fields when querying (including QA Contact field for verification assignee):
   ```
   fields: summary,status,priority,assignee,customfield_12310243,components,customfield_12315948
   ```

   **Note**: `customfield_12315948` is the "QA Contact" field, which identifies who will perform the verification work for OCPBUGS bugs.

2. Include these bugs in the sprint plan as verification work

3. **If no bugs found:**
   - Add informational note in the report that no OCPBUGS bugs in ON_QA were found for the specified component(s)

4. **If bugs found:**
   - Include in sprint plan with story points (if available)
   - Count toward total sprint capacity

### 🔧 Phase 3: Routine Quality Work Identification

**This phase identifies ongoing quality responsibilities that should be tracked in every sprint.**

Routine quality work represents recurring responsibilities that teams need to plan for in every sprint. This phase searches for three categories of work:

1. **Maintenance Release Ownership** - Search for unresolved issues related to z-stream or errata management:

   **If component is specified:**
   ```jql
   project = "{project-key}" AND
   component = "{component}" AND
   resolution = Unresolved AND
   (summary ~ "z-stream" OR summary ~ "errata") AND
   (summary ~ "lead" OR summary ~ "owner")
   ORDER BY priority DESC, updated DESC
   ```

   **If component is NOT specified:**
   ```jql
   project = "{project-key}" AND
   resolution = Unresolved AND
   (summary ~ "z-stream" OR summary ~ "errata") AND
   (summary ~ "lead" OR summary ~ "owner")
   ORDER BY priority DESC, updated DESC
   ```

2. **Root Cause Analysis (RCA) Ownership** - Search for unresolved issues related to RCA responsibilities:

   **If component is specified:**
   ```jql
   project = "{project-key}" AND
   component = "{component}" AND
   resolution = Unresolved AND
   (summary ~ "\"Root Cause Analysis\"" OR summary ~ "RCA") AND
   (summary ~ "lead" OR summary ~ "owner")
   ORDER BY priority DESC, updated DESC
   ```

   **If component is NOT specified:**
   ```jql
   project = "{project-key}" AND
   resolution = Unresolved AND
   (summary ~ "\"Root Cause Analysis\"" OR summary ~ "RCA") AND
   (summary ~ "lead" OR summary ~ "owner")
   ORDER BY priority DESC, updated DESC
   ```

3. **Daily Regression Monitoring** - Search for unresolved issues related to test monitoring and triage:

   **If component is specified:**
   ```jql
   project = "{project-key}" AND
   component = "{component}" AND
   resolution = Unresolved AND
   (summary ~ "\"daily regression\"" OR summary ~ "triage" OR summary ~ "monitor" OR summary ~ "\"rebase responsible\"" OR summary ~ "\"payload manager\"")
   ORDER BY priority DESC, updated DESC
   ```

   **If component is NOT specified:**
   ```jql
   project = "{project-key}" AND
   resolution = Unresolved AND
   (summary ~ "\"daily regression\"" OR summary ~ "triage" OR summary ~ "monitor" OR summary ~ "\"rebase responsible\"" OR summary ~ "\"payload manager\"")
   ORDER BY priority DESC, updated DESC
   ```

   **Important**: Include relevant fields when querying:
   ```
   fields: summary,status,priority,assignee,customfield_12310243,components
   ```

4. **For each category:**

   a. **If work items found:**
      - Include them in the sprint plan as routine quality work
      - Count story points toward sprint capacity (if available)
      - Note the assignee and status in the report

   b. **If no work items found:**
      - Add a **warning** to the sprint plan report
      - Recommend evaluating whether this type of routine work should be tracked
      - Include an action item to create an appropriate tracking issue if needed

5. **Deduplicate results**: If an issue matches multiple categories (e.g., "RCA lead for z-stream bugs"), only include it once but note which categories it covers.

### 🎯 Phase 4: Active Epic-Linked Work (HIGH PRIORITY)

**This phase identifies work linked to active epics using intelligent epic state analysis.**

1. **Find active epics in the target project**:

   Query for epics in "Dev Complete", "In Progress", or "Testing" state:
   ```jql
   project = "{project-key}" AND
   issuetype = Epic AND
   status IN ("Dev Complete", "In Progress", "Testing")
   ORDER BY priority DESC, created DESC
   ```

2. **For each epic, find linked tasks**:

   Find unresolved issues linked to this epic via either the `parent` field or the `Epic Link` custom field:

   **If component is specified:**
   ```jql
   project = "{project-key}" AND
   component = "{component}" AND
   (parent = "{epic-key}" OR "Epic Link" = "{epic-key}") AND
   resolution = Unresolved
   ORDER BY priority DESC, rank ASC
   ```

   **If component is NOT specified:**
   ```jql
   project = "{project-key}" AND
   (parent = "{epic-key}" OR "Epic Link" = "{epic-key}") AND
   resolution = Unresolved
   ORDER BY priority DESC, rank ASC
   ```

   **Important**: Always include `components`, `parent`, and Epic Link fields when querying:
   ```
   fields: summary,status,priority,assignee,customfield_12310243,components,parent,customfield_10014
   ```

   **Note**: The Epic Link custom field ID may vary by Jira instance. The example uses `customfield_10014`, which is common. See the "Epic Link Field" configuration section below to find your instance's field ID.

   **Multi-component handling**: Issues with multiple components (e.g., `["Frontend", "API", "Auth"]`) will be included if any of their components match the filter. In the report, display all components to provide full context.

3. **Additionally, find epics with stories in Review state**:

   a. Search for epics that have stories in "Review" state:
   ```jql
   project = "{project-key}" AND
   issuetype = Epic AND
   issueFunction in linkedIssuesOfRecursive("status = Review")
   ORDER BY priority DESC
   ```

   b. For each epic found, query for unresolved tasks linked to that epic via either the `parent` field or the `Epic Link` custom field:

   **If component is specified:**
   ```jql
   project = "{project-key}" AND
   component = "{component}" AND
   (parent = "{epic-key}" OR "Epic Link" = "{epic-key}") AND
   resolution = Unresolved
   ORDER BY priority DESC, rank ASC
   ```

   **If component is NOT specified:**
   ```jql
   project = "{project-key}" AND
   (parent = "{epic-key}" OR "Epic Link" = "{epic-key}") AND
   resolution = Unresolved
   ORDER BY priority DESC, rank ASC
   ```

4. **Deduplicate results**: Combine and deduplicate issues found in steps 2 and 3b to create the final list of epic-linked work.

5. **If NO tasks are linked to active epics**:
   - Add informational note to the report
   - Suggest reviewing if work should be linked to epics
   - List the active epic keys for reference

6. Mark these tasks as **HIGH PRIORITY** in the sprint plan

### 📋 Phase 5: Prioritized Backlog Selection

**This phase uses intelligent selection based on both priority and rank, while excluding items linked to inactive epics.**

1. Query the backlog for high-priority work:

   **If component is specified:**
   ```jql
   project = "{project-key}" AND
   component = "{component}" AND
   status IN ("To Do", "Backlog", "Selected for Development") AND
   priority IN (Blocker, Critical, Major)
   ORDER BY priority DESC, rank ASC
   ```

   **If component is NOT specified:**
   ```jql
   project = "{project-key}" AND
   status IN ("To Do", "Backlog", "Selected for Development") AND
   priority IN (Blocker, Critical, Major)
   ORDER BY priority DESC, rank ASC
   ```

   **Important**: Always include `components`, `parent`, and Epic Link fields when querying:
   ```
   fields: summary,status,priority,assignee,customfield_12310243,labels,components,parent,customfield_10014
   ```

   **Note**: The Epic Link custom field ID (`customfield_10014`) may vary by Jira instance. See the "Epic Link Field" configuration section below.

   **How ORDER BY works**:
   - `priority DESC`: Items are first grouped by priority (Blocker → Critical → Major)
   - `rank ASC`: Within each priority tier, items are ordered by rank (team's backlog ordering)
   - This ensures highest priority items come first, with team's rank determining order within each priority

   **Multi-component support**: When a component is specified, the query will match all issues that have that component, even if they have additional components. For example, filtering by `component = "Frontend"` will include issues with components like `["Frontend"]`, `["Frontend", "API"]`, or `["Frontend", "Auth", "Performance"]`.

2. **Filter out items linked to inactive epics**:

   For each backlog item returned from step 1:

   a. Check if the item has an epic link via either the `parent` field or the `Epic Link` custom field (e.g., `customfield_10014`)

   b. **If item has an epic link** (via `parent` OR Epic Link field):
      - Query the epic's status using the Jira API or MCP tool
      - Determine if epic is in an **inactive state** (NEW, PLANNING, TO DO, Backlog, etc.)
      - **If epic is inactive**: Exclude this item from sprint selection
      - **If epic is active** (In Progress, Dev Complete, Review, etc.): Include this item

   c. **If item has no epic link** (neither `parent` nor Epic Link field):
      - Include this item (it's available for sprint planning as a standalone backlog item)

   **Important**: Check BOTH the `parent` field and the Epic Link custom field, as different Jira configurations may use one or both for epic relationships.

   **Rationale**: Items linked to inactive epics should not be pulled into the current sprint because the epic work hasn't started yet. Only include:
   - Items with no epic link (standalone backlog items)
   - Items linked to active epics (where epic work is already underway)

3. Exclude duplicates already included in:
   - OCPBUGS ON_QA bugs (from Phase 2, if applicable)
   - Routine quality work (from Phase 3)
   - Epic-linked tasks (from Phase 4)

4. **If max story points specified:**
   - Calculate current story points from OCPBUGS bugs + routine quality work + epic-linked tasks
   - Select backlog items in priority and rank order until capacity reached
   - Stop when adding next item would exceed capacity

5. **If max story points NOT specified:**
   - Select top 20 backlog items by priority and rank

6. Extract story points from field: `customfield_12310243` (adjust if different in your Jira instance)

### 📊 Phase 6: Report Generation

Generate a comprehensive sprint plan report including:

1. **Sprint Summary**:
   - Component filter: Display "All Components" if no component specified, otherwise show the component name
   - Total issues selected
   - Breakdown by category (OCPBUGS ON_QA, routine quality work, epic-linked, backlog)
   - Story point summary vs capacity (if max-points specified)

2. **Categorized Issue Lists**:
   - **OCPBUGS in ON_QA** (verification work, if applicable)
     - Display QA Contact (from `customfield_12315948`) instead of assignee to show who will perform verification
     - Format: "QA Contact: [name]" or "No QA Contact assigned" if field is empty
   - **Routine Quality Work** (maintenance releases, RCA, daily regression monitoring)
   - **Epic-Linked Tasks** (high priority, organized by epic)
   - **Prioritized Backlog** (ordered by priority tier, then rank within each tier)

   **Multi-component display**: For issues with multiple components, display all components to provide full context. Format as a comma-separated list in brackets after the issue summary:
   - **PROJ-123** Issue summary `[Frontend, API, Auth]`
   - This helps identify cross-functional work and coordination needs

3. **Recommendations**:
   - Epic linkage suggestions for orphaned tasks
   - Capacity utilization analysis
   - **Warnings about missing routine quality work** (if any categories are empty)
   - Suggestions to create tracking issues for routine quality responsibilities
   - Information about items excluded due to inactive epic links (if applicable)
   - Next steps and action items

## Multi-Component Considerations

### How Multi-Component Filtering Works

When you specify a component filter (e.g., `Frontend`), the JQL query `component = "Frontend"` matches all issues that have "Frontend" as one of their components, regardless of how many other components are assigned.

**Examples:**
- Issue with `["Frontend"]` → **Matched** ✓
- Issue with `["Frontend", "API"]` → **Matched** ✓
- Issue with `["Frontend", "Auth", "Performance"]` → **Matched** ✓
- Issue with `["API", "Backend"]` → **Not matched** ✗

### Best Practices for Multi-Component Issues

1. **Display all components in reports**: Always show the full component list for each issue to provide context about cross-functional work

2. **Identify coordination needs**: Issues with multiple components often require coordination across teams. Flag these in your sprint planning discussions.

3. **Story point allocation**: For issues with multiple components (e.g., `["Frontend", "Backend"]`), story points are counted once in the sprint plan. Coordinate with other teams to avoid double-counting capacity.

### Report Format for Multi-Component Issues

The sprint plan report displays components for each issue as follows:

**Single component:**
- **PROJ-123** [Critical] Issue summary - 3 points

**Multiple components:**
- **PROJ-456** [Critical] Issue summary `[Frontend, API, Backend]` - 5 points
  - The component list in backticks helps identify cross-team dependencies

## Usage Examples

1. **Basic sprint planning (all components)**:
   ```
   /jira:plan-sprint MYAPP
   ```
   This will find all issues across all components in the MYAPP project.

2. **Sprint planning with specific component**:
   ```
   /jira:plan-sprint MYAPP Frontend
   ```
   This will find all issues with the Frontend component in the MYAPP project, including multi-component issues like those with `["Frontend", "API"]` or `["Frontend", "Auth", "Performance"]`.

3. **With story point capacity limit (all components)**:
   ```
   /jira:plan-sprint MYAPP --max-points 40
   ```

4. **With component and capacity limit**:
   ```
   /jira:plan-sprint MYAPP Backend --max-points 40
   ```

5. **Different project with component**:
   ```
   /jira:plan-sprint PLATFORM API
   ```

6. **With all options**:
   ```
   /jira:plan-sprint PLATFORM API --max-points 50
   ```

## Output Format

### Sprint Planning Report

**When component filter is specified:**
```markdown
# Sprint Plan
**Component**: Frontend | **Project**: MYAPP | **Capacity**: 45 story points
**OCPBUGS Components**: Microshift, Performance

---

## 📊 Sprint Summary

**Total Issues**: 25
- **OCPBUGS ON_QA**: 4 bugs (verification work)
- **Routine Quality Work**: 3 items (ongoing responsibilities)
- **Epic-Linked**: 8 tasks (HIGH PRIORITY)
- **Prioritized Backlog**: 10 items

**Story Points**: 47 / 45 (104% capacity - recommend reducing scope)
- Over capacity: 2 points

---

## 🐛 OCPBUGS in ON_QA (4 bugs)

These bugs need verification in this sprint:

1. **OCPBUGS-5678** [Critical] Application crash on startup `[Microshift]` - QA Contact: @alice - 2 points
2. **OCPBUGS-5679** [Major] Network connectivity issues `[Performance]` - QA Contact: @bob - 2 points
3. **OCPBUGS-5680** [Major] Storage volume mount failures `[Microshift]` - QA Contact: @alice - 1 point
4. **OCPBUGS-5681** [Normal] Configuration not persisted `[Performance]` - No QA Contact assigned - 1 point

---

## 🔧 Routine Quality Work (3 items)

Ongoing quality responsibilities that need attention this sprint:

**Maintenance Releases:**
1. **MYAPP-1100** [Normal] Z-stream 4.15.3 errata lead - Assigned to @alice - 2 points

**Root Cause Analysis:**
2. **MYAPP-1101** [Normal] RCA owner for customer escalations - Assigned to @bob - 1 point

**Daily Regression Monitoring:**
3. **MYAPP-1102** [Normal] Daily regression triage and payload manager - Assigned to @charlie - 2 points

---

## 🎯 Epic-Linked Tasks (8 tasks) - HIGH PRIORITY

Work linked to active epics in development:

**Epic: MYAPP-1000** "User Authentication Overhaul" (In Progress)
1. **MYAPP-1240** [Blocker] Implement OAuth2 integration `[Frontend, Auth]` - 5 points
2. **MYAPP-1241** [Critical] Add multi-factor authentication UI - 3 points
3. **MYAPP-1242** [Major] Session management improvements `[Frontend, Backend, Auth]` - 5 points

**Epic: MYAPP-1050** "Performance Optimization" (Dev Complete)
4. **MYAPP-1243** [Critical] Implement lazy loading for components - 3 points
5. **MYAPP-1244** [Major] Add React.memo optimization `[Frontend, Performance]` - 3 points
6. **MYAPP-1245** [Major] Reduce bundle size - 2 points

**Epic: MYAPP-1100** "Accessibility Improvements" (Review in Progress)
7. **MYAPP-1246** [Critical] ARIA labels for form inputs - 2 points
8. **MYAPP-1247** [Major] Keyboard navigation support - 3 points

**Note**: Issues with multiple components are shown with component list in backticks (e.g., `[Frontend, Auth]`). These often require cross-team coordination.

---

## 📋 Prioritized Backlog (10 items)

**Items are ordered by priority tier, then by rank (team's backlog ordering) within each tier.**

**Blocker Priority (1):**
- **MYAPP-1250** Critical security vulnerability fix `[Frontend, Security]` - 5 points

**Critical Priority (4):**
- **MYAPP-1252** API error handling improvements `[Frontend, API]` - 3 points
- **MYAPP-1253** Form validation refactor - 2 points
- **MYAPP-1254** Mobile responsiveness fixes `[Frontend, Mobile]` - 3 points
- **MYAPP-1255** User profile page redesign - 3 points

**Major Priority (5):**
- **MYAPP-1256** Documentation updates - 1 point
- **MYAPP-1257** Unit test coverage improvements `[Frontend, Testing]` - 2 points
- **MYAPP-1258** Component library updates - 2 points
- (Capacity limit reached - 2 additional Major items not included)

---

## 💡 Recommendations

1. **🐛 OCPBUGS workload**: 4 bugs (6 points) need verification
   - QA Contacts assigned: @alice (2 bugs), @bob (2 bugs)
   - 1 bug has no QA Contact assigned - needs assignment
   - Ensure QA Contact assignees have capacity allocated for verification work

2. **🔧 Routine quality work**: 3 items (5 points) identified
   - All routine quality responsibilities are tracked
   - Ensure assignees have capacity for ongoing work
   - Consider story point allocation for recurring responsibilities

3. **✅ Good epic alignment**: 8 tasks linked to 3 active epics in development
   - Epics are in active states (In Progress, Dev Complete, Testing)
   - All tasks properly organized under parent epics

4. **📊 Capacity utilization**: 104% (47/45 points)
   - Sprint is slightly over capacity by 2 points
   - Consider deferring lowest-priority backlog items or negotiating additional capacity

5. **🔄 Cross-functional coordination**: 5 issues have multiple components
   - Coordinate with Auth team on MYAPP-1240 `[Frontend, Auth]`
   - Coordinate with Backend team on MYAPP-1242 `[Frontend, Backend, Auth]`
   - Coordinate with Security team on MYAPP-1250 `[Frontend, Security]`
   - These issues may require additional planning or sync meetings

6. **🎯 Epic coverage**: All high-priority epics have associated sprint work
   - MYAPP-1000: 3 tasks (13 points)
   - MYAPP-1050: 3 tasks (8 points)
   - MYAPP-1100: 2 tasks (5 points)

7. **⏸️ Items excluded** (linked to inactive epics): 3 backlog items were excluded
   - 3 items are linked to epics in PLANNING or NEW state
   - These items should not be started until their parent epics become active
   - Review epic roadmap to understand when these items will be ready for sprint planning

---

## 📋 Next Steps

- [ ] Assign QA Contact for OCPBUGS bugs without QA Contact
- [ ] Ensure QA Contacts have capacity for OCPBUGS verification work
- [ ] Review routine quality work assignments and ensure capacity
- [ ] Review and confirm epic-linked tasks with Epic owners
- [ ] Reduce sprint scope by 2 points to match capacity
- [ ] Assign remaining tasks to team members
- [ ] Create sprint in Jira and add selected issues
- [ ] Schedule sync meetings for cross-functional work
- [ ] Coordinate with Epic owners on task priorities and dependencies
```

**When no component filter is specified (all components):**
```markdown
# Sprint Plan
**Component**: All Components | **Project**: MYAPP | **Capacity**: 45 story points
**OCPBUGS Components**: Microshift, Performance

---

## 📊 Sprint Summary

**Total Issues**: 35
- **OCPBUGS ON_QA**: 4 bugs (verification work)
- **Epic-Linked**: 15 tasks (HIGH PRIORITY)
- **Prioritized Backlog**: 16 items

**Story Points**: 68 / 45 (151% capacity - recommend reducing scope)
- Over capacity: 23 points

---

[Remaining sections follow same format as component-filtered report...]
```

**Example: When routine quality work is missing (warnings and action items):**

If the command doesn't find certain categories of routine quality work, the report will include warnings and action items:

```markdown
## 🔧 Routine Quality Work (1 item)

Ongoing quality responsibilities that need attention this sprint:

**Daily Regression Monitoring:**
1. **MYAPP-1102** [Normal] Daily regression triage and payload manager - Assigned to @charlie - 2 points

⚠️ **Missing routine quality work detected**

---

## 💡 Recommendations

...

8. **⚠️ Missing routine quality work**:
   - **No maintenance release tracking found**: No issues found for z-stream or errata leadership
     - Consider creating a tracking issue if your team manages maintenance releases
     - Example: "4.15.z z-stream lead" or "Errata owner for 4.15"

   - **No RCA ownership tracking found**: No issues found for Root Cause Analysis responsibilities
     - Consider creating a tracking issue if your team performs RCAs for escaped bugs
     - Example: "RCA owner for customer escalations" or "Root Cause Analysis lead"

   - These are recurring quality responsibilities that benefit from explicit tracking and story point allocation

---

## 📋 Next Steps

- [ ] Assign QA Contact for OCPBUGS bugs without QA Contact
- [ ] Ensure QA Contacts have capacity for OCPBUGS verification work
- [ ] ⚠️ **Evaluate need for maintenance release tracking issue**
- [ ] ⚠️ **Evaluate need for RCA ownership tracking issue**
- [ ] Review routine quality work assignments and ensure capacity
- [ ] Review and confirm epic-linked tasks with Epic owners
...
```

## Arguments

- **$1 – project-key** *(required)*
  The Jira project key to plan the sprint for.
  Example: `MYAPP`, `PLATFORM`, `USHIFT`

- **$2 – component** *(optional)*
  The component to filter sprint work by. When omitted, includes ALL issues across all components.
  When specified, this will match all issues that have this component assigned, including issues with multiple components.
  Example: `Frontend`, `API`, `Backend`, `QE`

  **Multi-component behavior**: Specifying `Frontend` will include issues with components like `["Frontend"]`, `["Frontend", "API"]`, or `["Frontend", "Auth", "Performance"]`.

  **Omit for all components**: If you want to plan a sprint across all components in the project, simply omit this argument.

- **--max-points** *(optional)*
  Maximum story points for the sprint to match team capacity.
  When specified, backlog selection will stop when capacity is reached.
  Example: `--max-points 40`

## Return Value
- **Markdown Report**: Comprehensive sprint plan with categorized work items, capacity analysis, and actionable recommendations

## Configuration Notes

### QA Contact Field (OCPBUGS)

The command uses the "QA Contact" field to identify who will perform verification work for OCPBUGS bugs: `customfield_12315948`

**If your Jira instance uses a different field:**
1. Use the Jira MCP tool to find the correct field:
   ```
   Use mcp__atlassian__jira_search_fields with keyword "qa contact"
   ```
2. Update the field reference in Phase 2 (OCPBUGS ON_QA Identification) to include your instance's QA Contact field ID
3. Update Phase 6 (Report Generation) to extract QA Contact from the correct field

**Why QA Contact instead of Assignee?**
- In OCPBUGS, the assignee is typically the developer who fixed the bug
- The QA Contact field identifies the QE team member who will verify the fix
- For sprint planning, teams need to know who will do the verification work, not who did the development

### Story Point Field

The command expects story points in field: `customfield_12310243`

**If your Jira instance uses a different field:**
1. Use the Jira MCP tool to find the correct field:
   ```
   Use mcp__atlassian__jira_search_fields with keyword "story"
   ```
2. Update the field reference when extracting story points in Phase 4

### Epic Link Field

The command checks for epic-issue relationships using BOTH:
- **`parent` field**: The modern standard field for parent-child relationships in Jira
- **Epic Link custom field**: A legacy custom field (commonly `customfield_10014`) used in older Jira configurations

**Finding your Epic Link custom field ID:**

1. Use the Jira MCP tool to search for the Epic Link field:
   ```
   Use mcp__atlassian__jira_search_fields with keyword "epic"
   ```

2. Look for a field named "Epic Link" in the results. The field ID will typically be something like `customfield_10014`, `customfield_10008`, or similar.

3. Update all references to the Epic Link field ID in the JQL queries if your instance uses a different ID:
   - Phase 4, step 2: Update the field list to include your Epic Link field ID
   - Phase 4, step 3b: Update the field list
   - Phase 5, step 1: Update the field list
   - Phase 5, step 2: Update the field reference when checking for epic links

**Why check both fields?**
- Modern Jira instances primarily use the `parent` field
- Older Jira instances may still use the Epic Link custom field
- Some instances in transition may have both
- Checking both ensures comprehensive epic-issue relationship detection

**Example field IDs by Jira type:**
- **Jira Cloud**: Often `customfield_10014` or `customfield_10008`
- **Jira Server/Data Center**: Varies by configuration, commonly `customfield_10100` or similar
- Always verify with your specific instance using the search method above

### Epic State Analysis

The command identifies active epics using the following criteria:

**Direct Epic States:**
- **"Dev Complete"**: Development is finished, implementation verification is the next priority
- **"In Progress"**: Active development ongoing, tasks should be prepared
- **"Testing"**: Epic is in testing phase, quality validation is ongoing

**Indirect Epic Detection (via linked stories):**
- Epics that have linked stories in **"Review"** state are considered active
- This captures epics where development work is being reviewed and validation will be needed soon

**Important Notes:**
- Epic states may vary by Jira project. Adjust the JQL queries in Phase 4 to match your project's workflow states.
- If your project uses different state names (e.g., "Ready for Testing" instead of "Dev Complete"), update the JQL in Phase 4, step 1.
- The `issueFunction in linkedIssuesOfRecursive("status = Review")` query requires the ScriptRunner plugin or similar advanced JQL support.

**Alternative Approach (if ScriptRunner is not available):**

If your Jira instance doesn't support `issueFunction`, use this manual two-step approach in Phase 4, step 3:

1. First, find all stories in Review state:
   ```jql
   project = "{project-key}" AND
   issuetype = Story AND
   status = Review
   ```

2. For each story found, check if it has a parent epic using the Jira API or by reading the `parent` field.

3. Collect the unique epic keys and query for unresolved tasks linked to those epics (same as Phase 4, step 3b).

### Inactive Epic States (Backlog Filtering)

The command excludes backlog items linked to epics in **inactive states** during Phase 5 (Prioritized Backlog Selection).

**Inactive Epic States** (items linked to these epics are excluded from sprint planning):
- **"NEW"**: Epic is newly created but not yet planned
- **"PLANNING"**: Epic is being planned but work hasn't started
- **"TO DO"**: Epic is queued but not yet active
- **"Backlog"**: Epic is in the backlog awaiting prioritization

**Active Epic States** (items linked to these epics may be included):
- **"In Progress"**: Active development ongoing
- **"Dev Complete"**: Development finished, verification needed
- **"Testing"**: Epic is in testing phase
- **"Review"**: Work being reviewed
- Any other state indicating active work

**Rationale**: Items linked to inactive epics should wait until the epic becomes active. This prevents:
- Starting work on epics that haven't been approved or prioritized
- Fragmenting work across too many concurrent initiatives
- Working on items before epic-level planning is complete

**Project-Specific Configuration**:
- Epic states vary by Jira project and workflow
- Adjust the inactive state list in Phase 5, step 2 to match your project's workflow
- Common variations: "New Feature", "Planned", "Scheduled", "Future", etc.

## See Also
- `jira:plan-qe-sprint` - QE-specific sprint planning with mandatory role validation
- `jira:grooming` - Backlog grooming meeting agendas
- `jira:status-rollup` - Sprint status rollup reports
- `jira:create` - Create Jira issues
