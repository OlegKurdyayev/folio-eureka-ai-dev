name: write-testrail-cases
description: Use when writing, generating, or adding manual test cases to TestRail. Produces structured cases with preconditions, step-by-step actions with expected results, and all required metadata fields. Also use when asked to cover a User Story with test cases, prepare cases for a new feature, or post cases to a TestRail section via API.
license: Apache-2.0
metadata:
  author: folio-org
  version: 1.2.0

---

# Write TestRail Cases

## Test Case Structure

Every test case must have these sections (in order):

**Metadata** — Type, Priority, Release, Test Group, References, and other classification fields.  
**Preconditions** — All users, their Capability Sets, required data setup, and system state before the test begins.  
**Steps** — Numbered actions with a concrete expected result for each step.

### Template

```
Title: [Action] [Object] [Condition]

Type:               Other
Priority:           [Critical | High | Medium | Low — choose based on impact]
Release:            Umbrellaleaf
Test Group:         [Smoke | Critical Path | Extended — choose based on scenario scope]
User Journey:       No
Multi-Tenant:       No
Bug Created:        No
Unstable:           No
ECS Enabled:        No
ECS Unsupported:    No
Capabilities Ready: Yes
AI Automated Test:  None
Execution Type:     Manual
Automation Scope:   None
Automated For:      None
Automation Team:    None
Dev Team:           [Value from "Development Team" field in the User Story]
References:         [Jira ticket IDs, comma-separated, e.g. MODFIN-273, UIF-657]

Preconditions:
- User A with following Capability Sets exists:
  - Data - [Module] [Resource] - [Action]
  - Procedural - [Module] [Resource] - [Action]
- User B (admin user with all FOLIO Capability Sets) is logged in
- [Required data: describe records, profiles, files, or system state that must exist before testing]
- User B is on [Starting app/page]

Steps:
1. Action:   [Actor] [verb] [object] on [location]
   Expected: [Observable UI or system result]

2. Action:   [Actor] [verb] [object]
   Expected: [Observable UI or system result]
```

## Title Format Rules

Use a plain descriptive sentence — no prefixes, no labels.

**Format:** `[Actor/Subject] [verb] [object] [condition]`

**Do's ✓**
- `User with Edit capability set can create locked mapping profile`
- `Mapping profile Status column shows Locked after lock checkbox is enabled`
- `Delete mapping profile modal shows all required elements`
- `Holdings export fails when submitted file contains invalid UUIDs`
- `Search filters mapping profiles by name and FOLIO record type`

**Don'ts ✗**
- ~~`Verify mapping profile creation`~~ — too vague, no subject or condition
- ~~`Negative: Holdings export with invalid UUIDs`~~ — no prefixes
- ~~`ECS | Delete locked mapping profile`~~ — no prefixes
- ~~`Load testing - Export 500k records`~~ — no prefixes
- ~~`Test that user can create profile`~~ — don't start with "Test that"


---

## Metadata Fields Reference

| Field | TestRail API key | Default | Options / Notes |
|---|---|---|---|
| Title | `title` | — | See Title format rules below. No prefixes (no "Verify", "Negative:", "ECS \|", "Load testing -") |
| Type | `type_id` | `Other` | Other, Acceptance, Accessibility, Automated, Compatibility, Destructive, Functional, Performance, Regression, Security, Smoke & Sanity, Usability |
| Priority | `priority_id` | — | Critical, High, Medium, Low — choose based on business impact |
| Release | `custom_release` | `Umbrellaleaf` | Use Umbrellaleaf unless story specifies another release |
| Test Group | `custom_test_group` | — | Smoke, Critical Path, Extended — see Test Group rules below |
| User Journey | `custom_user_journey` | `No` | No / Yes |
| Multi-Tenant | `custom_multi_tenant` | `No` | No / Yes |
| Bug Created | `custom_bug_created` | `No` | No / Yes |
| Unstable | `custom_unstable` | `No` | No / Yes |
| ECS Enabled | `custom_ecs_enabled` | — | Ask the user before each session: "Is this story ECS-related?" |
| ECS Unsupported | `custom_ecs_unsupported` | — | Ask the user before each session: "Is this feature unsupported in ECS?" |
| Capabilities Ready | `custom_capabilities_ready` | `Yes` | No / Yes |
| AI Automated Test | `custom_ai_automated_test` | `None` | None |
| Execution Type | `custom_execution_type` | `Manual` | Manual |
| Automation Scope | `custom_automation_scope` | `None` | None |
| Automated For | `custom_automated_for` | `None` | None |
| Automation Team | `custom_automation_team` | `None` | None |
| Dev Team | `custom_dev_team` | — | Take from "Development Team" field in the User Story |
| References | `refs` | — | Jira ticket IDs from the story, comma-separated |

---

## Test Group Rules

Choose based on the scenario's scope and criticality:

- **Smoke** — Core feature works at all. Use for the single most critical happy path that would immediately break the feature if it failed. Typically 1 case per feature area.
- **Critical Path** — Essential functionality users depend on daily. Create, edit, delete, key permissions and capability boundary checks, export/import flows, status transitions.
- **Extended** — Less frequent but important scenarios. Search and filtering, UI element verification, page title checks, edge cases, negative cases, load/performance checks, complex multi-step workflows.

Pattern observed in backlog: most cases are Critical Path or Extended. Smoke is rare (1-2 per feature area).

---

## Writing Guidelines

### Preconditions

All environments are Eureka. Always use Capability Sets — never legacy permissions.

- List all required users with their exact Capability Sets:
  ```
  Data - UI-Data-Export Settings - Edit
  Data - UI-Data-Export Settings Lock - Edit
  ```
- List required data that must exist before the test (profiles, records, files, UUIDs):
  ```
  - Unlocked mapping profile not referenced in any job profile exists
  - Locked mapping profile referenced in job profile exists
  - .csv file with Holdings UUIDs prepared
  ```
- List the starting system state (which app or page the user is on)
- Use named actors consistently: **User A** (restricted Capability Sets), **User B** (admin with all Capability Sets)

### Steps

**Do's ✓**
- One action per step — never combine two actions into one step
- Name the actor in every step: "User A clicks...", not just "Click..."
- Expected result must be a concrete, observable UI or system state
- For modal dialogs, always verify all visible elements (buttons, text, state)
- For list/table verifications, list the exact columns or elements expected
- For toast messages, include the exact text format with placeholders, e.g. `"The mapping profile <name> has been successfully deleted"`
- For multi-step flows within one logical action (e.g. open Actions menu, click Delete, confirm in modal), split into separate steps
- Cover all scenario types: happy path → edge cases → negative → capability boundaries

**Don'ts ✗**
- Don't write vague expected results like "works correctly" or "is successful"
- Don't skip expected results for intermediate navigation steps
- Don't bundle multiple user roles into one test case — one role per case
- Don't omit capability boundary scenarios when the story involves roles or restrictions
- Don't use legacy permission names — always use Capability Sets
- Don't combine UI element verification with action steps

### Step patterns from the backlog

**Navigating to a page:**
```
1. Action:   User B navigates to Settings > Data export > Field mapping profiles
   Expected: "Field mapping profiles" pane is displayed with "New" button, search box, and table with profiles
```

**Opening an Actions menu:**
```
2. Action:   User B clicks "Actions" menu on the mapping profile view form
   Expected: Menu expands and displays the following options: Edit, Duplicate, Delete
```

**Confirming a modal:**
```
3. Action:   User B clicks "Delete" button on "Delete mapping profile" modal
   Expected: Modal closes, toast message "Mapping profile <name> has been successfully deleted" is displayed, profile is removed from the list
```

**Verifying table columns:**
```
4. Action:   User B verifies columns in the "Field mapping profiles" table
   Expected: Table includes the following columns: Name, FOLIO record type, Format, Updated, Updated by, Status
```

**Verifying "Status" field after lock/unlock:**
```
5. Action:   User B checks the "Status" column for the newly created locked profile
   Expected: "Locked" is displayed in the "Status" column
```

**File export flow:**
```
1. Action:   User B clicks "or choose file" button on the Jobs pane and submits .csv file with UUIDs
   Expected: "Select job profile to run the export" modal opens with list of job profiles, search box, and disabled "Search" button

2. Action:   User B clicks on "Default holdings export job profile"
   Expected: "Are you sure you want to run this job?" modal opens with dropdown list, "Cancel" button (enabled), "Run" button (disabled)

3. Action:   User B selects "Holdings" from the dropdown list
   Expected: "Run" button becomes enabled

4. Action:   User B clicks "Run" button
   Expected: Job starts; progress bar is visible in "Running" section; after completion, "Logs" pane displays new row with .mrc file as hyperlink, "Completed" status, correct counts in Total/Exported/Failed columns
```

---

## Scenario Coverage Checklist

For every User Story, generate cases that cover:
- [ ] Primary happy path (valid data, expected flow, correct UI state)
- [ ] UI element verification (buttons, labels, columns, counts — especially for new pages)
- [ ] Edit and duplicate flows (where applicable)
- [ ] Delete flow including: Cancel button, confirm delete, verify removal from list
- [ ] Capability boundaries: restricted user cannot perform actions outside their Capability Sets
- [ ] Negative cases: invalid input, wrong job profile, empty files, referenced profiles blocking deletion
- [ ] Edge cases: repeated UUIDs, profiles referenced in multiple job profiles
- [ ] Status field changes (e.g. locked vs unlocked → "Locked" vs empty Status)
- [ ] Toast messages: verify exact text and format
- [ ] Multi-tenant behavior (if applicable to the story)
- [ ] Load/performance (if story involves bulk operations — mark as Extended)

---

## Quick Reference Checklist

- [ ] Title follows `[Action] [Object] [Condition]` pattern and is unique
- [ ] All metadata fields are filled with correct defaults or story-derived values
- [ ] `Dev Team` taken from "Development Team" field in the User Story
- [ ] `refs` contains all Jira ticket IDs from the story
- [ ] Preconditions use Capability Sets only — no legacy permissions
- [ ] Preconditions describe all required data (records, files, profiles) that must exist
- [ ] Each step has exactly one action and one concrete expected result
- [ ] Toast messages include exact text with `<name>` placeholders where applicable
- [ ] Modal dialogs verified with all their elements (text, buttons, states)
- [ ] Happy path, UI verification, edit/duplicate/delete, capability boundaries covered
- [ ] Named actors used consistently throughout (User A / User B)
- [ ] No vague expected results ("works", "is correct", "is successful")

---

## Context Files (Business Logic Reference)

The repository contains context files at `skills/context/` that describe the business logic of each application area. Always read the relevant context file before generating test cases — it provides domain knowledge needed to write accurate preconditions, realistic data setups, and correct expected results.

### Step 1 — Ask the user which application area they are testing

Before reading any User Story or generating cases, ask:

> _"Which application area does this story belong to? I'll load the relevant context file."_

Provide a numbered list of available areas based on files present in `skills/context/`. For example:

```
1. Data Export (job profiles, mapping profiles, export flows)
2. Inventory (instances, holdings, items)
3. Finance (acquisition units, funds, budgets)
4. Settings & Permissions (capability sets, roles)
5. [other areas as files are added]
```

Wait for the user's answer before proceeding.

### Step 2 — Read the corresponding context file

Map the user's answer to the file in `skills/context/`:

| Area | File |
|---|---|
| Data Export | `skills/context/data-export.md` |
| Inventory | `skills/context/inventory.md` |
| Finance | `skills/context/finance.md` |
| Settings & Permissions | `skills/context/settings-permissions.md` |
| *(add rows as new files are created)* | |

Read the file silently. Use it to:
- Understand the domain objects involved (e.g. what a "locked profile" means, what "referenced in job profile" implies)
- Determine realistic precondition data (what records must exist and in what state)
- Identify business rules that should translate into test scenarios (e.g. locked profile → only Duplicate available in Actions menu)
- Validate that generated expected results match actual application behavior

### Step 3 — Proceed with User Story and case generation

After reading the context file, read the User Story and generate cases as described below.

If a context file for the requested area does not exist yet, notify the user:
> _"I don't have a context file for this area yet. I'll generate cases based on the User Story alone — results may be less accurate for domain-specific preconditions."_

---

## TestRail API Integration

### Credentials

Read credentials automatically from:
```
/Users/Oleg_Kurdyayev/stripes-testing/OK
```
Do not ask the user for credentials — read this file silently before every API call.

### Workflow

1. Ask the user: _"Is this story ECS-related? Should ECS Enabled be set to Yes?"_ and _"Is this feature unsupported in ECS? Should ECS Unsupported be set to Yes?"_ — set fields based on the answer.
2. Read the full User Story and all acceptance criteria.
3. Take `Dev Team` from the "Development Team" field in the User Story.
4. Take `refs` from all Jira ticket IDs mentioned in the story.
5. Identify distinct scenarios using the coverage checklist above.
6. Generate one test case per scenario.
7. Ask the user: _"Please provide the TestRail Section (folder) ID where the cases should be posted."_
8. Show a preview before posting:
   ```
   Ready to post 5 test cases to Section ID: XXXXX

   1. User with Edit capability set can create unlocked mapping profile — Other / High / Critical Path
   2. User with Edit capability set can edit unlocked mapping profile — Other / High / Critical Path
   3. User with Edit capability set cannot delete mapping profile referenced in job profile — Other / High / Critical Path
   4. Mapping profile "Status" column shows "Locked" after lock checkbox is enabled — Other / Medium / Extended
   5. User with View-only capability set cannot see "New" or "Actions" buttons — Other / High / Critical Path

   Confirm? (yes / no / edit)
   ```
9. Wait for explicit confirmation, then post.

### Add case request

```
POST /index.php?/api/v2/add_case/{section_id}
Content-Type: application/json
Authorization: Basic <base64(email:api_key)>
```

```json
{
  "title": "User with Edit capability set can delete unlocked mapping profile not referenced in job profile",
  "type_id": 1,
  "priority_id": 2,
  "refs": "MODFIN-273",
  "custom_preconds": "- User B with following Capability Sets exists:\n  - Data - UI-Data-Export Settings - Edit\n- User B is logged in\n- Unlocked mapping profile NOT referenced in any job profile exists\n- User B is on Settings > Data export > Field mapping profiles",
  "custom_steps_separated": [
    {
      "content": "User B clicks on the row with the unlocked mapping profile from Preconditions",
      "expected": "Mapping profile view form is displayed; \"Lock profile\" checkbox is disabled, unchecked; \"Actions\" menu is enabled"
    },
    {
      "content": "User B clicks \"Actions\" menu",
      "expected": "Menu expands and displays the following options: Edit, Duplicate, Delete"
    },
    {
      "content": "User B clicks \"Delete\" option",
      "expected": "\"Delete mapping profile\" modal opens with: \"The mapping profile <name> will be deleted.\" text, \"Cancel\" button (enabled), \"Delete\" button (enabled, focused)"
    },
    {
      "content": "User B clicks \"Cancel\" button",
      "expected": "\"Delete mapping profile\" modal closes; mapping profile view form is displayed"
    },
    {
      "content": "User B clicks \"Actions\" menu > \"Delete\" > \"Delete\" in modal",
      "expected": "Modal closes; toast message \"Mapping profile <name> has been successfully deleted\" is displayed; \"Field mapping profiles\" pane shows list without deleted profile"
    }
  ],
  "custom_release": "Umbrellaleaf",
  "custom_test_group": "Critical Path",
  "custom_execution_type": "Manual",
  "custom_multi_tenant": 0,
  "custom_user_journey": 0,
  "custom_bug_created": 0,
  "custom_unstable": 0,
  "custom_ecs_enabled": 0,
  "custom_ecs_unsupported": 0,
  "custom_capabilities_ready": 1,
  "custom_automation_scope": "None",
  "custom_automated_for": "None",
  "custom_automation_team": "None",
  "custom_ai_automated_test": "None"
}
```

### Success output

After posting all cases, report:
```
✅ Posted 5 test cases to Section 1042:
  C123456 — User with Edit capability set can create unlocked mapping profile
  C123457 — User with Edit capability set can edit unlocked mapping profile
  C123458 — User with Edit capability set cannot delete mapping profile referenced in job profile
  C123459 — Mapping profile Status column shows Locked after lock is enabled
  C123460 — User with View-only capability set cannot see New or Actions buttons
```

### Error handling

| HTTP status | Action |
|---|---|
| `401` | Report credentials error — check file `/Users/Oleg_Kurdyayev/stripes-testing/OK` |
| `400` | Show the full error body and identify which field caused it |
| `404` | Confirm the Section ID exists and is accessible |
| Any failure | Do not skip silently — report the failed case title and full error message |

---

## Fallback Output (no API access)

If posting via API fails entirely, output each case in this format:

```
---
Title:          [title]
Type:           Other
Priority:       [High | Medium | ...]
Release:        Umbrellaleaf
Test Group:     [Smoke | Critical Path | Extended]
Execution Type: Manual
Dev Team:       [from story]
References:     [jira_ids]

Preconditions:
- User A with following Capability Sets exists:
  - [Capability Set 1]
- User B (admin with all FOLIO Capability Sets) is logged in
- [Required data that must exist]
- User B is on [app/page]

Steps:
1. Action:   [step]
   Expected: [result]

2. Action:   [step]
   Expected: [result]
---
```

For Capability Sets reference, see the `write-user-story` skill.  
For API field IDs, run: `GET /index.php?/api/v2/get_case_fields`
