name: write-testrail-cases
description: Use when writing, generating, or adding manual test cases to TestRail. Produces structured cases with preconditions, step-by-step actions with expected results, and all required metadata fields. Also use when asked to cover a User Story with test cases, prepare cases for a new feature, or post cases to a TestRail section via API.
license: Apache-2.0
metadata:
  author: folio-org
  version: 1.3.0

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
Test Group:         [Smoke | Critical Path | Extended — required]
User Journey:       No
Multi-Tenant:       No
Bug Created:        No
Unstable:           No
ECS Enabled:        No   [set to Yes only when the story explicitly tests ECS/cross-tenant behavior]
ECS Unsupported:    No   [fixed default — never ask]
Capabilities Ready: Yes
Execution Type:     Manual   [required]
Dev Team:           [Value from "Development Team" field in the User Story]
Customer Name:      [All | MOBIUS/GALILEO | LOC — only if the story targets a specific customer]
Labels:             AI   [always added to every case]
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
- `User can create a multi-select custom field` ← always include actor ("User can...")
- `User can check out requested item in Central tenant` ← ECS case: no prefix needed, tenant context is in the title

**Don'ts ✗**
- ~~`Verify mapping profile creation`~~ — too vague, no subject or condition
- ~~`Create a text field custom field`~~ — missing actor; rewrite as `User can create a text field custom field`
- ~~`Delete custom fields`~~ — missing actor; rewrite as `User can delete custom fields`
- ~~`Verify that final custom field can be removed`~~ — don't start with "Verify that"; rewrite as `User can remove the last remaining custom field`
- ~~`Negative: Holdings export with invalid UUIDs`~~ — no prefixes
- ~~`ECS | Delete locked mapping profile`~~ — no pipe-style prefixes
- ~~`Load testing - Export 500k records`~~ — no prefixes
- ~~`Test that user can create profile`~~ — don't start with "Test that"

> **ECS title exception:** You may use `[ECS <Area>]` prefix only when the test is exclusively ECS-specific and the area label adds essential disambiguation — e.g. `[ECS Loans] User can check out requested item in Central tenant`. Use sparingly; prefer embedding the tenant context in the title sentence itself.

---

## Metadata Fields Reference

> **Resolve IDs at runtime.** Dropdown fields (Type, Priority, Release, Test Group, Execution Type, Dev Team, Customer Name, Automation Scope/Team/For) take a **numeric option ID**, not the label string — sending a label returns `400 Field :<name> is not a valid natural number`. Look up the current IDs with `GET /index.php?/api/v2/get_priorities`, `get_case_types`, and `get_case_fields`. Checkbox fields take a JSON **boolean** (`true` / `false`). The option IDs below are a snapshot from `foliotest.testrail.io` and may change — always confirm via the API.
>
> **Required fields:** `custom_test_group` and `custom_automation_type` (labelled "Execution Type"). A case will not post without them.
>
> **Standing defaults (do not ask the user):** `ECS Unsupported = false` on every case, and **every case gets the "AI" label**. Set `ECS Enabled = true` only when the story explicitly tests cross-tenant / ECS behavior.

| Field | TestRail API key | Type | Default | Options / Notes (current IDs) |
|---|---|---|---|---|
| Title | `title` | string | — | Plain descriptive sentence; no prefixes ("Verify", "Negative:", "ECS \|", "Load testing -") |
| Type | `type_id` | dropdown (id) | `Other` | Resolve via `get_case_types`. Current: **Other=7** |
| Priority | `priority_id` | dropdown (id) | — | Resolve via `get_priorities`. Current: Low=1, Medium=2, High=3, Critical=4 |
| Release | `custom_release` | dropdown (id) | latest release | Current latest: **R2 2026 Umbrellaleaf=21**, R1 2026 Trillium=20, R1 2025 Sunflower=19 |
| Test Group | `custom_test_group` | dropdown (id) | — | **Required.** Smoke=1, Critical Path=2, Extended=3, Obsolete=4, Draft=5, Backend=6, Edge Cases=7 |
| Execution Type | `custom_automation_type` | dropdown (id) | `Manual` (2) | **Required.** Automated=1, Manual=2, Karate=3, Unit=4, Backend Component=5 (TestRail label is "Execution Type"; API key is `custom_automation_type`) |
| User Journey | `custom_user_journey` | checkbox (bool) | `false` | `true` / `false` |
| Multi-Tenant | `custom_multi_tenant` | checkbox (bool) | `false` | `true` / `false` |
| Bug Created | `custom_bug_created` | checkbox (bool) | `false` | `true` / `false` |
| Unstable | `custom_unstable` | checkbox (bool) | `false` | `true` / `false` |
| ECS Enabled | `custom_ecs_enabled` | checkbox (bool) | `false` | Set to `true` when the story explicitly tests ECS/cross-tenant behavior. Otherwise `false`. |
| ECS Unsupported | `custom_ecs_unsupported` | checkbox (bool) | `false` | **Always `false`. Never ask the user.** |
| Capabilities Ready | `custom_capabilities_ready` | checkbox (bool) | `true` | `true` / `false` |
| Dev Team | `custom_dev_team` | dropdown (id) | — | From "Development Team" in the story. e.g. Concorde=1, Firebird=3, Folijet=4, Spitfire=6, Thor=7, **Thunderjet=8**, Vega=9, Volaris=13, Citation=18, Eureka=21 (full list via `get_case_fields`) |
| Customer Name | `custom_customer_name` | dropdown (id) | — | Optional. All=1, MOBIUS/GALILEO=2, LOC=3 — set only when the story targets a specific customer |
| Automation Scope | `custom_automation_scope` | dropdown (id) | — | Optional; omit if N/A. Review by MQA=1, Review by PO=2, Automation Ready=3, Not an automation candidate=4, Mriya scope=5, Karate Approved=6, Karate Not Applicable=7, FE scope=8 (no "None" option) |
| Automated For | `custom_case_automated_in` | dropdown (id) | — | Optional; omit if N/A. Old releases=1, R2 2024 Ramsons=2, R1 2025 Sunflower=3, R2 2025 Trillium=4 |
| Automation Team | `custom_automation_team` | dropdown (id) | — | Optional; omit if N/A. TaaS=1, Mriya=2, FE team=3 (no "None" option) |
| References | `refs` | string | — | Jira ticket IDs from the story, comma-separated |
| Labels | `labels` | array | `["AI"]` | **Always add the "AI" label to every case.** Pass label ID(s) — resolve via `get_labels/{project_id}` (on `foliotest.testrail.io`, "AI" = id **67**) — or the title string `"AI"`. |

Other optional fields present in the instance (normally omitted): `custom_ai_model`, `custom_ai_type`, `custom_backlog_tech_dept` (checkbox), `custom_goals`, `custom_mission`, `custom_expected`, `custom_testrail_bdd_scenario`. The skill's former `custom_execution_type`, `custom_automated_for`, and `custom_ai_automated_test` keys do **not** exist — use `custom_automation_type` and `custom_case_automated_in` instead.

---

## Test Group Rules

Choose based on the scenario's scope and criticality:

- **Smoke** — Core feature works at all. Use for the single most critical happy path that would immediately break the feature if it failed. Typically 1 case per feature area.
- **Critical Path** — Essential functionality users depend on daily. Create, edit, delete, key permissions and capability boundary checks, export/import flows, status transitions.
- **Extended** — Less frequent but important scenarios. Search and filtering, UI element verification, page title checks, edge cases, negative cases, load/performance checks, complex multi-step workflows.

Pattern observed in backlog: most cases are Critical Path or Extended. Smoke is rare (1-2 per feature area).

---

## Writing Guidelines

> **Core principle:** Write every test case as if it will be executed by a QA engineer who has never seen this feature before. Steps must be precise enough that any team member — Firebird, Spitfire, Vega, Volaris, Thunderjet — can execute the case without guessing. Use the same level of detail regardless of team or area: same step granularity, same precondition format, same expected result specificity.

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
- Use named actors consistently: **User** — the actor performing the test
- Always include specific monetary amounts (e.g. Allocated = 100.00), fund relationships (Transfer to/from), and ledger settings when they affect test behavior
- **For ECS cases:** always specify which tenant each data record belongs to (Central or Data tenant name), which service points are configured, and what affiliation the user starts with. Example:
  ```
  - Active Ledger exists in Central tenant for the current fiscal year
  - Fund A (Allocated = 200.00) exists in Data tenant (College), related to the Ledger above
  - User B is logged in with affiliation set to Central tenant
  - Item with barcode 12345 exists in Data tenant, status Available
  - Pickup service point SP-Central is configured in Central tenant
  ```

### Steps

**Do's ✓**
- One action per step — never combine two actions into one step
- Name the actor in every step: "User A clicks...", not just "Click..."
- Expected result must be a concrete, observable UI or system state
- When an expected result contains **two or more distinct observable items**, format them as a bullet list, not a single sentence. Example:Expected: - Modal closes
- Toast message "Order <PO number> was successfully closed" is displayed
- Order status changes to "Closed"
- For modal dialogs, always add a dedicated step that lists ALL visible UI elements (dropdowns, fields, buttons and their states — enabled/disabled). Example: `"Move allocation" modal opens and contains: "From" dropdown, "To" dropdown, "Amount" field, "Cancel" button (enabled), "Confirm" button (disabled)`
- For list/table verifications, list the exact columns or elements expected
- For toast messages, always include the exact text with placeholders, e.g. `"<amount> was successfully allocated to the budget <Fund-FY>"`. Never write just "success toast is displayed"
- When a modal has interactive buttons that change form state (e.g. Swap, Recalculate, Calculate), always add a dedicated scenario covering that button — verify that fields, error messages, and button states update correctly after clicking it
- For multi-step flows within one logical action (e.g. open Actions menu, click Delete, confirm in modal), split into separate steps
- Cover all scenario types: happy path → edge cases → negative → capability boundaries
- When a step has important caveats or warnings that affect test execution (e.g. a field may show validation errors that need to be resolved before proceeding), add a `NOTE:` line inside the expected result. Example: `Boxes filled with "\" appear in "008" field. NOTE: Some dropdowns of "008" field may be highlighted in red — select valid values from those dropdowns before saving.`
- **For ECS cases:** after every action in one tenant, add a verification step in the other tenant if the story requires cross-tenant consistency. Always name the tenant explicitly in both the action and expected result.

**Don'ts ✗**
- Don't write vague expected results like "works correctly" or "is successful"
- Don't skip expected results for intermediate navigation steps
- Don't bundle multiple user roles into one test case — one role per case
- Don't omit capability boundary scenarios when the story involves roles or restrictions
- Don't use legacy permission names — always use Capability Sets
- Don't combine UI element verification with action steps
- Don't bundle multiple actions into one step even in ECS cases — tenant switching is its own step with its own expected result
- Don't use `→` inside a single step to chain two actions — split into separate steps. Example: ~~`Click "Actions" → Select "+ New MARC bib record"`~~ → split into two steps
- Don't use `|` as a title separator — ~~`Create | Verify that MARC bib can be opened`~~ → rewrite as `User can open MARC bib record created with linked field`

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

**Navigation with arrow (Settings):**
```
1. Action:   User B navigates to Settings -> Users -> Custom fields
   Expected: "Custom fields" pane is open with list of existing custom fields and "Edit" button
```
> Always use `>` as the navigation separator when generating steps, regardless of what style other teams use. Example: `Settings > Inventory > Call number browse`.

**Inline edit in table (pencil icon):**
```
1. Action:   User B clicks the "Edit" (pencil) icon next to the "Call numbers (all)" row in the table
   Expected: "Call number types" column of that row switches to an enabled multi-select dropdown; "Cancel" button (enabled) and "Save" button (disabled) appear in the "Actions" column

2. Action:   User B clicks "Cancel" button in the "Actions" column
   Expected: Row returns to read-only state; no changes are saved
```

**Multi-select dropdown verification:**
```
1. Action:   User B clicks on the multi-select dropdown in the "Call number types" column
   Expected: Dropdown expands and displays all available options in alphabetical order; each option has a checkbox; no options are pre-selected

2. Action:   User B selects options one by one from the multi-select dropdown
   Expected: Each selected option appears in the input field of the multi-select element; dropdown remains open after each selection; "Save" button becomes enabled after at least one option is selected

3. Action:   User B removes all selected options from the multi-select dropdown
   Expected: Input field of multi-select dropdown is empty; "Save" button becomes disabled again
```

**Drag and drop (reordering):**
```
1. Action:   User B hovers over the drag handle (six-dots icon) next to the custom field record
   Expected: "Change custom field order" tooltip appears; cursor changes to grab

2. Action:   User B drags the custom field record to a new position in the list
   Expected: Custom field moves to the new position; order is updated in the list
```

**Multi-tab testing (edge case — use only when the story explicitly tests concurrent tab behavior):**
```
1. Action:   User B duplicates the current browser tab while the edit form is open
   Expected: Edit form opens in the second tab showing the same state as the first tab

2. Action:   User B saves changes in the first tab
   Expected: Changes saved successfully in the first tab

3. Action:   User B switches to the second tab and attempts to save different changes
   Expected: [expected conflict/success behavior per story requirements]
```

**DevTools / Network verification (edge case — use only when the story requires verifying API response or hidden fields not visible in UI):**
```
1. Action:   User B opens browser DevTools (F12) and navigates to the Network tab
   Expected: DevTools pane is open; Network tab is active and recording requests

2. Action:   User B performs the action that triggers the relevant API call (e.g. clicks "Custom fields")
   Expected: Network tab captures the /custom-fields request; response body contains [expected field/value]; UI does NOT display [hidden field]
```

**Tenant switching (ECS cases):**
```
1. Action:   User B switches affiliation to Central tenant
   Expected: Active tenant changes to Central; page reloads showing Central tenant context

2. Action:   User B switches affiliation to Data tenant (College)
   Expected: Active tenant changes to Data tenant (College); page reloads showing Data tenant context
```

**Barcode scanning — patron (Circulation):**
```
1. Action:   User B opens "Check out" app and enters patron barcode on the "Scan patron card" pane, then clicks "Enter"
   Expected: Patron scanned; patron name and details displayed on the left pane
```

**Barcode scanning — item (Circulation):**
```
2. Action:   User B enters item barcode on the "Scan items" pane and clicks "Enter"
   Expected: Item scanned; loan created; item row appears in the scanned items list with status "Checked out"
```

**Accordion expansion:**
```
1. Action:   User B expands "Loans" accordion on the user details page
   Expected: "Loans" accordion expands and displays "# open loans" and "# closed loans" hyperlinks

2. Action:   User B clicks "# closed loans" hyperlink
   Expected: "Loans" page opens with "Closed loans" tab active; loan from preconditions is displayed in the list
```

**Cross-tenant verification (ECS cases):**
```
1. Action:   User B switches affiliation to Data tenant and opens Loan details page for Item 1 via Circulation log
   Expected: Loan details page opened in Data tenant; renewal action displayed in the actions list; due date matches the due date from the Central tenant loan
```

**Search & filter pane verification:**
```
1. Action:   User B clicks "View all" button in the "Logs" pane
   Expected: Page updates with "Search & filter" pane on the left and "Logs" main pane; "Search & filter" pane contains:
             - "ID" dropdown
             - Search box
             - "Search" button (disabled)
             - "Reset all" button (disabled)
             - "Errors in export" accordion (expanded)
             - "Started running" accordion (collapsed)
             - "Ended running" accordion (collapsed)
             - "Job profile" accordion (collapsed)
             - "User" accordion (collapsed)
```
> Always list all filter accordions with their initial expanded/collapsed state. List Search and Reset buttons with their initial enabled/disabled state.

**Pagination verification:**
```
1. Action:   User B scrolls to the bottom of the results table
   Expected: Paginator is displayed below the table; page navigation controls are visible; total records count matches the number shown in the header
```

**Date range filter (date picker):**
```
1. Action:   User B expands "Started running" accordion in the "Search & filter" pane
   Expected: "Started running" accordion expands and displays "From" and "To" date/time fields

2. Action:   User B fills in "From" date field with a start date and "To" date field with an end date, then clicks "Apply"
   Expected: Results table filters to show only records within the specified date range; records count updates accordingly
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
- [ ] Toast messages: verify exact text with placeholders — never just 'success toast'
- [ ] Modal element verification: dedicated step listing all visible UI elements with their states on first open
- [ ] Interactive modal buttons (Swap, Recalculate): dedicated scenario verifying fields, errors, and button states update correctly after clicking
- [ ] Multi-tenant behavior (if applicable to the story)
- [ ] Load/performance (if story involves bulk operations — mark as Extended)
- [ ] **ECS cases:** cross-tenant consistency — action in Central tenant verified in Data tenant and vice versa
- [ ] **ECS cases:** tenant switching steps are explicit with named tenant in action and expected result
- [ ] Drag and drop / reordering flows (if the story involves changing order of records)
- [ ] Multi-tab behavior (only if the story explicitly covers concurrent editing in multiple tabs)
- [ ] Inline edit flows: verify Save/Cancel appear inline, Save disabled until change made, row returns to read-only on Cancel
- [ ] Multi-select dropdowns: verify all options shown, dropdown stays open after selection, Save enables/disables correctly
- [ ] Search & filter pane: verify all accordions present with correct initial expanded/collapsed state; Search/Reset buttons disabled by default
- [ ] Pagination: verify paginator appears when results exceed page limit; total count matches header
- [ ] Date range filters: verify "From"/"To" fields appear on accordion expand; results filter correctly after applying

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
- [ ] **ECS cases:** `ECS Enabled = true` set in metadata; tenant specified in every cross-tenant step

---

## Context Files (Business Logic Reference)

The repository contains context files at `references/context/` that describe the business logic of each application area. Always read the relevant context file before generating test cases — it provides domain knowledge needed to write accurate preconditions, realistic data setups, and correct expected results.

### Step 1 — Ask the user which application area they are testing

Before reading any User Story or generating cases, ask:

> _"Which application area does this story belong to? I'll load the relevant context file."_

Provide a numbered list of available areas based on files present in `references/context/`. For example:

```
1. Data Export (job profiles, mapping profiles, export flows)
2. Inventory (instances, holdings, items)
3. Finance (acquisition units, funds, budgets)
4. Settings & Permissions (capability sets, roles)
5. [other areas as files are added]
```

Wait for the user's answer before proceeding.

### Step 2 — Read the corresponding context file

Map the user's answer to the file in `references/context/`:

| Area | File |
|---|---|
| Data Export | `references/context/data-export.md` |
| Inventory | `references/context/inventory.md` |
| Finance | `references/context/finance.md` |
| Settings & Permissions | `references/context/settings-permissions.md` |
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
~/.folio-credentials
```
Do not ask the user for credentials — read this file silently before every API call.

### Workflow

1. Read the full User Story and all acceptance criteria.
2. Take `Dev Team` from the "Development Team" field in the User Story.
3. Take `refs` from all Jira ticket IDs mentioned in the story.
4. **Before generating any cases — perform a Scenario Analysis:**
   - Read the story and identify ALL distinct scenarios using the coverage checklist
   - Present the list to the user in this format:
   ```
   I've read the story. Here are the scenarios I plan to cover:

   Happy path:
   1. User can successfully [main action] with valid data → Critical Path
   2. [other happy path scenarios]

   UI verification:
   3. [page/modal] shows all required elements on open → Extended

   Capability boundaries:
   4. User without [Capability Set] cannot [action] → Critical Path

   Negative / edge cases:
   5. [action] fails when [condition] → Extended
   6. [edge case] → Extended

   Does this coverage look complete? Any scenarios to add or remove before I write the cases?
   ```
   - Wait for the user's confirmation or corrections before proceeding
5. Generate one test case per confirmed scenario. Set `ECS Enabled = true` only if the story tests ECS/cross-tenant behavior; always set `ECS Unsupported = false`; always add the **"AI"** label.
6. Ask the user: _"Please provide the TestRail Section (folder) ID where the cases should be posted."_
7. Show a preview before posting:
   ```
   Ready to post 5 test cases to Section ID: XXXXX

   1. User with Edit capability set can create unlocked mapping profile — Other / High / Critical Path
   2. User with Edit capability set can edit unlocked mapping profile — Other / High / Critical Path
   3. User with Edit capability set cannot delete mapping profile referenced in job profile — Other / High / Critical Path
   4. Mapping profile "Status" column shows "Locked" after lock checkbox is enabled — Other / Medium / Extended
   5. User with View-only capability set cannot see "New" or "Actions" buttons — Other / High / Critical Path

   Confirm? (yes / no / edit)
   ```
8. Wait for explicit confirmation, then post.

### Add case request

```
POST /index.php?/api/v2/add_case/{section_id}
Content-Type: application/json
Authorization: Basic <base64(email:api_key)>
```

```json
{
  "title": "User with Edit capability set can delete unlocked mapping profile not referenced in job profile",
  "type_id": 7,
  "priority_id": 3,
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
  "custom_release": 21,
  "custom_test_group": 2,
  "custom_automation_type": 2,
  "custom_dev_team": 4,
  "custom_customer_name": 1,
  "custom_multi_tenant": false,
  "custom_user_journey": false,
  "custom_bug_created": false,
  "custom_unstable": false,
  "custom_ecs_enabled": false,
  "custom_ecs_unsupported": false,
  "custom_capabilities_ready": true,
  "labels": [67]
}
```

> The dropdown values above are **option IDs**, not labels: `custom_release` 21 = "R2 2026 Umbrellaleaf", `custom_test_group` 2 = "Critical Path", `custom_automation_type` 2 = "Manual", `custom_dev_team` 4 = "Folijet", `custom_customer_name` 1 = "All". Resolve these per instance via `get_case_fields` / `get_priorities` / `get_case_types`. Optional dropdowns (Automation Scope/Team/For, AI Model/Type) are omitted when not applicable. `custom_ecs_unsupported` is **always `false`**; `custom_ecs_enabled` is `true` only for ECS stories. `"labels": [67]` is the **"AI"** label, always added to every case (resolve the ID via `get_labels/{project_id}`).

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
| `401` | Report credentials error — check file `~/.folio-credentials` |
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
