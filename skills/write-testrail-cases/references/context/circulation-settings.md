# FOLIO Circulation Settings — Business Logic Context

> Built: 2026-07-21 · Refreshed: 2026-11-24 (full merge of the 2026-11-07 delta) · Release: Umbrellaleaf R2 2026 (in progress; ui-circulation 13.0.0 IN_PROGRESS, mod-circulation 24.6.0-SNAPSHOT, mod-circulation-storage 17.6.0-SNAPSHOT — previous: Trillium R1 2026 = ui-circulation 12.0.x) · Builder: build-app-context (refresh mode)
> Sources (2026-11-24): GitHub folio-org/ui-circulation (master 43dc24c: `translations/ui-circulation/en.json`, `package.json`, `src/index.js`, `src/settings/Validation/*`, `CHANGELOG.md`); folio-org/mod-circulation (`descriptors/ModuleDescriptor-template.json`, `NEWS.md`, `doc/circulationrules.md`); folio-org/mod-circulation-storage (`descriptors/ModuleDescriptor-template.json`); folio-org/folio-integration-tests (`mod-circulation`, `mod-circulation-storage` Karate); folio-org/stripes-testing (`cypress/e2e/settings/circulation/*`); Jira UICIRC (Done/Closed since 2024-06, ~195 issues scanned) + CIRC (settings-related, ~115 scanned); TestRail suite 21 sections 102/103/104/52794/72019/169/1535/1132/1144; Confluence "Circulation - For use at location - Trillium - Notes on functionality", "User-specific loan anonymization"; docs.folio.org source `folio-org/docs content/en/docs/Settings/Settings_circulation/settings_circulation.md`.
> UI strings: `en.json` is the source of truth; `en_US.json` (Lokalise copy) lags by 37 keys (automatic renewals, text-message notice format, staff slip names, user-specific anonymization, notice-policy reorder) — do not assert those from `en_US.json`.

This file covers **Settings > Circulation**; the runtime circulation apps have their own files (check-in.md, check-out.md, loans.md, requests.md, fees-fines.md). Patron notice *templates and policies* are split into `patron-notices.md`.

---

## What is Circulation Settings

`Settings > Circulation` configures how circulation behaves: **policies** (loan, request, overdue fine, lost item fee, patron notice), **circulation rules** that map patron-group × material-type × loan-type × location → those policies, plus fixed due date schedules, staff slips, request cancellation reasons, title-level-request config, loan anonymization, print settings and check-in/out "Other settings". Everything the Check out / Check in / Requests / Loans / Fees-fines apps do at runtime is driven by these settings.

### Navigation (exact labels — en.json `settings.index.*`, routes from `src/index.js`)

| Group (`settings.index.*`) | Pages (label → route → view permission) |
|---|---|
| `General` | `Circulation rules` → `rules` → `ui-circulation.settings.view-circulation-rules`; `Other settings` → `checkout` → `…view-other-settings`; `Staff slips` → `staffslips` → `…view-staff-slips`; *(ECS requests enabled only)* `Title level requests (TLR)` → `title-level-requests`, `Consortium title level requests (TLR)` → `consortium-title-level-requests` |
| `Loans` | `Fixed due date schedules` → `fixed-due-date-schedules`; `Loan anonymization` → `loan-anonymization` (perm `…loan-history`); `Loan policies` → `loan-policies` |
| `Fee/fine` | `Overdue fine policies` → `fine-policies`; `Lost item fee policies` → `lost-item-fee-policy` |
| `Patron notices` | `Patron notice policies` → `notice-policies`; `Patron notice templates` → `patron-notices`; *(ECS requests enabled)* `TLR patron notice templates` → `tlr-patron-notice-templates` |
| `Requests` | `Request cancellation reasons` → `cancellation-reasons`; `Request policies` → `request-policies`; `Title level requests` (non-ECS only) ; `Print hold requests` → `print-hold-requests` (perm `…view-staff-slips`); `View print details` → `view-print-details` (perm `…request-print-details`) |

Pane title: `Circulation` (`settings.index.paneTitle`). **TLR placement differs by tenant type**: with ECS requests enabled, TLR lives under **General** as `Title level requests (TLR)` and its notice templates are a separate page `TLR patron notice templates`; otherwise a single `Title level requests` page sits under **Requests** (`src/index.js`).

---

## Circulation rules (the mapping engine)

`Settings > Circulation > General > Circulation rules` — pane `Circulation rules editor` (`settings.circulationRules.paneTitle`), a text editor mapping **criteria** to **policies**. Criteria pickers: `Patron groups`, `Material types`, `Loan types`, `Institutions`/`Campuses`/`Libraries`/`Locations` (location tree pickers `Select institution`/`Select campus`/`Select library`/`Select location`, `Done`), keywords `ANY`, `All`, `! (not)`. Policy pickers: `Loan policies`, `Request policies`, `Patron notice policies`, `Overdue fine policies`, `Lost item fee policies`. Save toast: `Rules were successfully updated.` (`settings.circulationRules.rulesUpdated`). Filter field: `filter rules`.

### Exact editor grammar [confirmed — C651, C654, C401724; mod-circulation `doc/circulationrules.md`]

- **Criteria letters** before `:`: `g` patron group, `m` material type, `t` loan type, `a` institution, `b` campus, `c` library, `s` location. Typing at a new line auto-suggests these; selecting one shows a value picker populated from that attribute's configured records.
- **Policy letters** after `:`: `l` loan, `r` request, `n` patron notice, `o` overdue fine, `i` lost item fee — any order, **exactly one of each type** (circulationrules.md "Policies"), e.g. `m DVD : l one-hour r allow-all n send-no-notices o overdue-fine-policy i lost-item-fee-policy`.
- Incomplete rule on Save: `Must contain one of each policy type, missing type [next expected policy type]` (C654 — not present in ui-circulation en.json; it is a backend parse error, assert only as observed in C654).
- Other malformed syntax (missing comma between values, missing policies, wrong indentation, criteria conjunction without `+`) blocks Save with an error above the line (C654). Example backend 422 body: `{"message": "no viable alternative at input '…'", "line": 3, "column": 41}` (CIRC-2445).
- **Criteria combination**: `+` concatenates criteria on one line; indentation nests a child rule under a parent (circulationrules.md "Criteria").
- **Comments**: `#` **or** `/` starts a line comment (circulationrules.md "Line comment"; docs.folio.org "Circulation rule tips"). `#` lines also act as section titles matched by the `filter rules` box; the filter treats `* ? \ [` literally (C401724).
- **Allowed name characters**: names referenced in rules may contain only `a-z A-Z 0-9 -` (circulationrules.md) — the editor inserts codes/ids for picked values.
- **`fallback-policy:`** line defines the default for all five policy types (circulationrules.md; C650).
- On Save, a "Last updated" stamp (date + user) appears top-left of the editor (C651/C654 observation).
- **Shadow (DCB) locations** are selectable in the location pickers of the rules editor only (ui-circulation 11.0.4, `includeShadow`/`includeShadowLocations`; UICIRC-1271). They must not appear in other location dropdowns (Settings > Tenant locations, holdings/items, orders — UICIRC-1271 QA comment).

### Rule priority when several rules match [source — mod-circulation `doc/circulationrules.md` "Multiple matching rules"]

A `priority:` line (before the first rule) lists up to three regulations checked in order; the last must be `first-line` or `last-line`:
1. `criterium (t, s, c, b, a, m, g)` — each rule is ranked by its **highest-priority criterium type**; rules with the highest wins (with the default order, any rule containing `t` beats a rule with only `m`/`g`).
2. `number-of-criteria` — more criterium types wins; any number of location types (`a b c s`) count as one; `all` keyword lets a type count without restricting names.
3. `first-line` / `last-line` — tie-break by position.
Legacy form `priority: t, s, c, b, a, m, g` = `criterium(t, s, c, b, a, m, g), number-of-criteria, last-line`.

---

## Policies

### Loan policy [confirmed — en.json, C638–C648, C1554460–C1554463]
Accordions `General information`, `Loans`, `Renewals`, `Request management`. Key fields: `Loan policy name`, `Description`, `Loanable`, `Loan profile` (`Fixed` / `Rolling`), `Loan period`, `Fixed due date schedule` / `Fixed due date schedule (due date limit)`, `Closed library due date management`, `Opening time offset`, `Grace period`, `Item limit`, `For use at location` + `Hold shelf expiration period`. Renewals: `Renewable`, `Unlimited renewals`, **`Automatic renewals`**, `Number of renewals allowed`, `Renew from` (`System date` / `Current due date`), `Renewal period different from original loan`. Request management: recalls (`Recall return interval`, `Minimum guaranteed loan period for recalled items`, `Alternate recall return interval for overdue loans`, `Alternate grace period for recalled items`, `Allow recalls to extend due dates for overdue loans`), holds (`Alternate loan period at checkout for items with an active, pending hold request`, `Allow renewal of items with an active, pending hold request`, `Alternate loan period at renewal for items with an active, pending hold request`) — labels from en.json `settings.requestManagement.*` (ui-circulation master, 2026-11-26).
- Delete: `Delete loan policy`; message `{name} will be {removed}.`; in-use guard `Cannot delete Loan policy` / `This policy cannot be deleted, as it is in use by one or more records.` (original build, github).
- **Validation (ui-circulation `src/settings/Validation/loan-policy/loans.js`)**: when Loanable — `Loan profile` and `Closed library due date management` required; Rolling — `Loan period` duration required, integer > 0 (`The value must be integer and greater than 0`) and interval required (`Please make a selection`); Fixed — `Fixed due date schedule` required; `Opening time offset` required/integer > 0 only when the closed-library option uses it; `Item limit` → `Item limit must be > 0, =< 9999 or blank` (always validated). Generic required text: `Please fill this in to continue`.
- **Automatic renewals** (UICIRC-1351/1387/1390/1399; ui-circulation 13.0.0; storage `loan-policy-storage` 2.5):
  - Checkbox `Automatic renewals` shown directly under `Unlimited renewals`, only while `Renewable` is checked; unchecking `Renewable` hides all renewal fields incl. Automatic renewals (UICIRC-1351, UICIRC-1390 Sc.1).
  - Info popover: `Automatic renewals are available for loans with intervals of one day or longer.` (`settings.loanPolicy.automaticRenewals.infoPopover`).
  - When checked: required `Attempt first renewal` with radios `On due date` (default) / `day(s) before due date` (+ number), then optional `Attempt additional retries for` `day(s)` (UICIRC-1390 Sc.3).
  - `day(s) before due date`: integer 1–100 → `The value must be integer and must be between 1 and 100`; empty → `Please fill this in to continue` (UICIRC-1390 Sc.4–5). Retries: integer 0–100 or blank → `The value must be integer and must be between 0 and 100 or blank`; blank is stored as 0 (UICIRC-1351 notes).
  - BE fields: `renewalsPolicy.automaticRenewal`, `firstAutomaticRenewalAttemptDaysBeforeDueDate`, `automaticRenewalRetryPeriodInDays`; absent `automaticRenewal` = disabled; 0 and 100 are valid boundaries (CIRC-2687).
  - Nightly job eligibility (CIRC-2584, status Blocked — not yet released): loan eligible only if policy has automatic renewal on, loan is inside the attempt window, normal renewal conditions pass and no active hold/recall blocks; blocked loans get Circulation log entries (mod-circulation NEWS 24.6.0).

### Request policy [confirmed — en.json, C410745, C410746, C411717]
`Request policy name`, `Description`, `Request types allowed` (Page/Hold/Recall); per type `Allow all pickup service points` / `Allow some pickup service points`. Hold info popover: `If the "Fail to create title level hold when request is blocked by circulation rule" setting is unchecked, title level holds will always be allowed. (Settings > Circulation > Title level requests)` (`settings.requestPolicy.policyTypes.hold.infoPopover.content`; C411717). Errors: `A request policy with this name already exists`, `Please select at least one service point`. Delete guard: `Request policy cannot be deleted` / `This request policy is used by circulation rules and it cannot be deleted`.
- Allowed pickup service points must be pickup locations: POST/PUT with a non-pickup or non-existent SP → 422 `One or more Pickup locations are no longer available`, code `INVALID_ALLOWED_SERVICE_POINT`; UI shows the same text (Karate `mod-circulation-storage/.../request-policies.feature`; Cypress C410745).
- Selected SPs persist after page reload (C410746).
- `+ New` opens with defaults — it must not inherit settings of a previously edited policy (UICIRC-1293 fix).

### Overdue fine policy [confirmed — en.json; C5557, C3613]
`Overdue fine policy name`, `Overdue fine` (amount per interval), `Count closed days/hours/minutes`, `Maximum overdue fine`, `Forgive overdue fine if item renewed`, `Overdue recall fine`, `Ignore grace periods for recalls`, `Maximum recall overdue fine`, **Reminder fees** (sequence, interval, frequency, fee, notice method/template, `Ignore grace periods for recalls`/`…for holds`, `Clear patron block when paid`, `Allow renewal of items with reminder fee(s)`).
Validations (en.json `settings.finePolicy.validate.*`): `Maximum overdue fine must be greater than or equal to Overdue fine`; `Maximum overdue recall fine must be greater than or equal to Overdue recall fine`; `Overdue fine must be greater than 0 if interval set`; `Overdue recall fine must be greater than 0 if interval set`; `Please enter an overdue fine amount or set maximum overdue fine to 0`; `Please enter an overdue recall fine amount or set maximum overdue recall fine to 0`. Delete: `Delete overdue fine policy`.

### Lost item fee policy [confirmed — en.json; C5558, C196744]
`Lost item fee policy name`, `Items aged to lost after overdue`, `Patron billed after aged to lost`, `Recalled items aged to lost after overdue`, `Patron billed for recall after aged to lost`, `Charge amount for item` (`Actual cost` / `Set cost`), `Lost item processing fee` + `Charge lost item processing fee if item declared lost by patron` / `…aged to lost by system`, `Replacement allowed`, `Replacement processing fee`, return/renew refund settings, `No fees/fines shall be refunded if a lost item is returned more than … late`. Delete: `Delete lost item fee policy`.
Cross-field validations (en.json `settings.lostItemFee.validate.*`), verbatim:
- `'Lost item processing fee' cannot be > 0 unless 'Charge lost item processing fee if item declared lost by patron ?' and/or 'Charge lost item processing fee if item aged to lost by system ?' is 'Yes'`
- `'Lost item processing fee' must be > 0 and 'Items aged to lost after overdue' amount must be > 0 for this selection`
- `'Replacement processing fee' cannot be > 0 unless 'Replacement allowed' is 'Yes` (sic — closing quote missing in en.json)
- `Entry required if 'Patron billed after aged to lost' amount entered`; `Entry required if 'Patron billed for recall after aged to lost' amount entered`; `Please enter amount`; `Required if there is a possibility of no fee/fine being charged for a lost item`
- Fractional processing fee (0 < x < 1, e.g. 0.25) is valid (UICIRC-1249).

### Fixed due date schedules [confirmed — en.json; C641, C367970, C1594647–C1594650]
`Fixed due date schedule name`, `Description`, rows `Date range {number}` with `Date from` / `Date to` / `Due date`. Create title `Create fixed due date schedule`. Validations: `Date range {num1} cannot overlap with date range {num2}`, `To date must be after from date`, `Due date must be on or after to date`, `At least one schedule must be entered`, `Please enter a unique name.` Delete guard `Schedule cannot be deleted when used by one or more loan policies.`; success `The fixed due date schedule {name} was successfully {deleted}.` Saving an edit without changing the name must succeed (C367970).
- Time zone: in America/Vancouver, due dates after 2026-11-01 must not shift by one hour; schedules saved before the moment-timezone update keep the old offset until re-saved (C1594647–C1594650; UICIRC-1391, FOLIO-4580; see vancouver timezone incident).

### Request cancellation reasons [confirmed — en.json]
`Cancel reason`, `Description (internal)`, `Description (public)`. Delete: `Delete cancel reason` → `The cancel reason <b>{term}</b> will be <b>deleted.</b>` → `The cancel reason <b>{term}</b> was successfully <b>deleted</b>`; in use → `Cannot delete the cancel reason` / `This cancel reason cannot be deleted, as it is in use by one or more records.` Duplicate names are rejected (Cypress `cannot-duplicate-cancellation-reason.cy.js`).

---

## Closed Library Due Date Management — exact resulting due date [confirmed — C829891, C829892, C831956, C844247–C844249, C844877, C844878, C1621466]

Short-term = minutes/hours, long-term = days/weeks/months (docs.folio.org "Loan policies"). Verify via Loan details "Due date":
- `Keep the current due date/time` (short) / `Keep the current due date` (long): due date unchanged even if the SP is closed (C829891, C844247).
- Short, `Move to the end of the current service point hours`: due date truncated EARLIER to the end of the current day's open hours (C829892). **For short-term periods > 24 h** (e.g. 48 h ending on a closed Saturday, open Friday in between) it moves to the end of the last open day before the closed day (C1621466, CIRC-2578).
- Short, `Move to the beginning of the next open service point hours`: pushed LATER to the next opening, **plus `Opening time offset`** (offset 1 minute + opening 11:00 → 11:01) (C831956). Also applies with a Fixed profile (C844877).
- Long, `Move to the end of the previous open day`: moves BACKWARD to 11:59 PM of the closest prior open day (C844248); with a Fixed schedule as well (C844878).
- Long, `Move to the end of the next open day`: moves FORWARD to 11:59 PM of the next open day (C844249 — resolves the former inference).

---

## Floating Collections [C605980, C605981, C605982]

A Location can be flagged "Floating enabled" and tied to service points.
- Check-in at an SP whose location has floating enabled: item → `Available`; Temporary AND Effective location → that floating location (C605980).
- Check-in at an SP without a floating location: item → `In transit`, locations unchanged (C605981).
- **An open request overrides floating**: item → `In transit to {pickup SP}`, no relocation (C605982).

## "For Use At Location" (reading-room circulation) [C870003, C869999, C1307935, C916267, C870000–C870002, C916248–C916251, C916268; Confluence Trillium notes]

**Deployment switch**: mod-circulation env var `ENABLE_FOR_USE_AT_LOCATION` must be `true`; default `false` (CIRC-2457). On environments without it the loan-policy checkbox exists but the workflow is inactive — always state the env flag in Preconditions.

**Setup**:
1. Loan policy with `For use at location` checked → requires `Hold shelf expiration period` in day(s)/week(s)/month(s) only; expiry time defaults to 11:59 PM, so minutes/hours do not work (Confluence notes; UICIRC-1239).
2. A loan type (e.g. "For use at location") and/or patron group, mapped by a circulation rule to that policy (C1307935).
3. Settings > Tenant > Service points > `Default check-in action for use at location` (C869999).

**Check-in**: options `Keep on hold shelf` / `Close loan and return item` / `Ask for action` (default from the SP; not sticky in the app). `Ask for action` opens modal `For use at location` with body `Item {title} ({material type}) (Barcode: {barcode}) is for use at location at {service point}.` and buttons `Cancel`, `Close loan and return item`, `Keep on hold shelf` (C916267).
- `Keep on hold shelf`: loan stays open, status `Checked out`, circ action `Held`, hold shelf expiration reset.
- `Close loan and return item`: loan closes; item → `Available` / `In transit` / `Awaiting pickup`; circ action `Check in` (Confluence notes).
- Circ actions: `In use` (checked out), `Held` (on hold shelf) — Users app shows `Use at location` / `Held: n` / `In use: n` (C870003, C870000).
- Requests: New request shows loan type and a `Use at location` indicator; `Delivery` fulfilment is not offered (C916249, C916251).
- **Hold-shelf expiration is capped at the loan due date**; on renewal it extends again (CIRC-2681, NEWS 24.6.0 — In Review; recall case still open in a separate ticket).

---

## Other Settings [confirmed — en.json `settings.checkout.*`; C1385309, UICIRC-1316/1317/1331/1333]

- `Patron id(s) for checkout scanning`: `Barcode`, `External system ID`, `FOLIO record number (ID)`, `Username`, `User custom fields` (+ custom-field multiselect). At least one required → `Please select to continue`; `User custom fields` checked with no field selected → `Please make a selection`, not saved (UICIRC-1316, UICIRC-1331).
  - `Save` is disabled when the form returns to the saved state (check then uncheck) and when the required warning is shown (UICIRC-1316; C1385309).
  - Re-checking `User custom fields` after saving it unchecked starts with an empty selection (UICIRC-1317).
  - An already selected custom field is greyed in the dropdown and is removed when clicked; removing any chip removes exactly that one (UICIRC-1333).
  - No custom fields defined → `No matching custom fields have been configured`.
- `Users custom fields to display at Check out` (UICIRC-1091) — with default settings Check out must not show all custom fields (UICHKOUT-1012).
- `Enable audio alerts` (`Yes`/`No`) + `Select audio-alerts theme`: `Classico` / `Moderna` / `Futura`.
- `Automatically end check in and check out session after period of inactivity` + `Timeout duration` `minutes` → `Please enter a whole number greater than zero to continue`.
- `Perform wildcard lookup of items by barcode in circulation apps (Check in, Check out)`.
- Values persist through mod-circulation `/circulation/settings` (setting name `other_settings`) since UICIRC-1247 (Trillium).

## Staff slips [en.json `settings.staffSlips.*`; section 1144 (63 cases)]

- Fixed set of slips, names now localized via `settings.staffSlips.names.*`: `Hold`, `Transit`, `Transit (mediated requests)`, `Pick slip`, `Request delivery`, `Search slip (Hold requests)`, `Due date receipt` (UICIRC-1354). The stored `name` must not be changed via API — printing looks templates up by stored name and prints empty pages otherwise (UICIRC-1354).
- Form: `Staff slip name`, `Description`, `Template content`, `Maintained as raw HTML?` (shown when `staff-slips-storage` ≥ 1.1; UICIRC-1226), `Preview` / `Print`; token groups `Item`, `Effective location`, `Staff slip`, `Borrower`, `Loan`, `Request`, `Request delivery address`, `Requester`.
- Preview renders fixed sample values per token; `{{staffSlip.Name}}` always previews as `Transit` regardless of slip — expected behaviour, closed as Won't Do (CIRC-2496). Do not write a case expecting the real slip name in Preview.
- Delete message: `The staff slip <strong>{name}</strong> will be <strong>deleted</strong>.`

## Loan anonymization [en.json `settings.loanHistory.*`; C9217–C9235; docs.folio.org]

- Library policy (`Library loan anonymization policy`) → `Anonymize closed loans`: `Never` / `Immediately after {name} closes` / `Retain for a set duration after {name} closes` (+ `Length`, `Select interval`); validation `Please enter a whole number greater than zero to continue`, `Please select to continue`.
- `Treat closed loans with associated fee/fines differently` → `Anonymization policy for loans with fees/fines` → `Anonymize closed loans with associated fees/fines`: `Never` / `Immediately after {name} closes (or is suspended)` / `Retain for a set duration after {name} closes (or is suspended)`; `Exception for payment method` + `Add exception` (`Waive`/`Transfer`/`Replace`/`Pay`; duplicate → `This payment method is already selected`). Verbiage changed in UICIRC-1200.
- **User-specific loan anonymization** (UICIRC-1361 — status In QA; CIRC-2698/2699 Closed; Confluence "User-specific loan anonymization"): section `User-specific loan anonymization` with `Do not allow user-specific loan anonymization` (default) / `If a user requests anonymization as early as possible, then anonymize closed loans` (immediately / retain for duration). Popover text per `settings.loanHistory.userSpecific.infoPopover`. Users get `According to library policy` (default) / `As early as possible` / `Never` in the user record (UIU-3595).
  - Job policy per loan: feature off → library policy; no preference / user not found → library policy; `As early as possible` → "earliest" policy (library policy if earliest not configured); `Never` → never anonymized; job keeps running even when the library policy is `Never` (CIRC-2698; Confluence).
  - Fee/fine loans: timed only by the shared fee/fine rule from the last fee/fine close; while a fee/fine is open the loan is not anonymized (Confluence).
  - Disabling the feature reverts to library policy; a warning dialog is planned (Confluence — text not in en.json yet).
- Scheduler: hourly, up to 50,000 loans per run (`SCHEDULED_ANONYMIZATION_NUMBER_OF_LOANS_TO_CHECK`) (CIRC-2459 comment). "X after loan closes" counts from system return date; loans closed without check-in are anonymized only under `Immediately` (Confluence).

## Title level requests, print settings

- `Allow title level requests`, `"Create title level request" selected by default`, `Fail to create title level hold when request is blocked by circulation rule`, `Notice templates` (`Confirmation notice` / `Cancellation notice` / `Expiration notice`, `Select notice`; "If no template is selected, no notice will be sent."). Disabling while TLRs are open → modal `Cannot change "Allow title level requests"` / `"Allow title level requests" cannot be changed because it is in use by one or more requests`. Unchecked `Fail to create…` ⇒ TLR holds always allowed (docs.folio.org; en.json popover).
- A view-only user must not see an active `Save` on the TLR page (UICIRC-1130).
- **Consortium TLR** (ECS): `Enable consortium title level requests (TLR)`; checking it shows `Exclude tenants` multiselect, unchecking hides it (UICIRC-1270). While base TLR is off: notification `Consortium title level requests (TLR) is disabled. The setting will become active after the "Allow title level requests" checkbox is checked and saved in the title level requests (TLR…` (en.json `settings.consortiumTlr.notification`, truncated here).
- `Print hold requests`: `Allow print hold requests (Open - Not yet filled)` (enables Search slips). `View print details`: `Enable view print details (Pick slips)`; disabling opens `Disable view print details (Pick slips)` / `Selecting Confirm will stop the recording of print job details.` (`Confirm`/`Cancel`); success `Setting was successfully updated.`, error `Something went wrong.` Backend flag: circulation setting `printEventLogFeature` (`enablePrintLog`, boolean or string — CIRC-2448).

---

## API surface [source — ModuleDescriptor-template.json of mod-circulation / mod-circulation-storage]

| Endpoint | Methods | permissionsRequired |
|---|---|---|
| `/circulation/rules` | GET / PUT | `circulation.rules.get` / `circulation.rules.put` |
| `/circulation/rules/{loan,request,notice,overdue-fine,lost-item}-policy` (+ `-all`) | GET | `circulation.rules.<type>.get` (lost-item-all: `circulation.rules.lost-item-all.get`) |
| `/circulation-rules-storage` | GET / PUT | `circulation-storage.circulation-rules.get/put` |
| `/circulation/settings[/{id}]` (interface `circulation-settings` 1.0) | GET, POST, PUT, DELETE | `circulation.settings.collection.get`, `.item.get/post/put/delete` |
| `/circulation-settings-storage/circulation-settings[/{id}]` | CRUD | `circulation-storage.circulation-settings.*` |
| `/loan-policy-storage/loan-policies[/{id}]` (2.5) | CRUD + collection DELETE | `circulation-storage.loan-policies.*` |
| `/request-policy-storage/request-policies[/{id}]` | CRUD | `circulation-storage.request-policies.*` |
| `/patron-notice-policy-storage/patron-notice-policies[/{id}]` (0.13) | CRUD | `circulation-storage.patron-notice-policies.*` |
| `/fixed-due-date-schedule-storage/fixed-due-date-schedules[/{id}]` | CRUD | `circulation-storage.fixed-due-date-schedules.*` |
| `/staff-slips-storage/staff-slips[/{id}]` (1.1) | CRUD | `circulation-storage.staff-slips.*` |
| `/cancellation-reason-storage/cancellation-reasons[/{id}]` | CRUD | `circulation-storage.cancellation-reasons.*` |
| overdue / lost item fee policies (mod-feesfines) | CRUD | `overdue-fines-policies.*`, `lost-item-fees-policies.*` (ui `package.json`) |

**Settings storage migration (Trillium, UICIRC-1247/1324)**: tenant settings moved from mod-configuration/mod-settings to `/circulation/settings` entries by `name`: `other_settings` (former CHECKOUT/other_settings), `loan_history` (LOAN_HISTORY), `TLR`, `PRINT_HOLD_REQUESTS`, `printEventLogFeature`; ECS TLR (`generalTlr`/`regularTlr`) still reads mod-settings scope `circulation` (TLR view perm includes `mod-settings.global.read.circulation`). Free-form JSON values — no schema change needed for new keys (CIRC-2698 comment).

## Capability Sets (Eureka)

UI permission sets (display names, `package.json`), each `view` + `create, edit and remove` pair: `Settings (Circulation): Can view all circulation settings`; `… Can view loan policies` / `… Can create, edit and remove loan policies`; `… Can view notice policies` / `… notice policies`; `… Can view patron notice templates` / `… patron notice templates`; `… Can view request policies` / `… request policies`; `… Can view overdue fine policies` / `… overdue fine policies`; `… Can view lost item fee policies` / `… lost item fee policies`; `… Can view fixed due date schedules` / `… fixed due date schedules`; `… Can view staff slips` / `… Can edit staff slips` / `… Can create, edit and remove staff slips`; `… Can view cancellation reasons` / `… cancellation reasons`; `… Can view other settings` / `… other settings`; `… View circulation rules` / `… Can create, edit and remove circulation rules`; `… Can view loan history` / `… Can edit loan history`; `… Title level request view` / `… Title level request edit`; `… Can enable request print details`.
Eureka capability sets confirmed by the Cypress dictionary (`cypress/support/dictionary/capabilitySets.js`, display form = `<Type> - <resource> - <Action>`): `Settings - UI-Circulation Settings Loan-Policies - Manage`, `Settings - UI-Circulation Settings Other-Settings - Manage`. Confirmed in Jira UICIRC-1356 (spelled there with lowercase `data`/action; type prefix shown as `Data`, not verified in the UI): `Data - UI-Circulation Settings Staff-Slips - Manage`, `Data - UI-Circulation Settings Request-Print-Details - View`.
Every other circulation-settings capability set (rules, request/fine/lost-item/notice policies, fixed due date schedules, cancellation reasons, loan anonymization, TLR, view-* variants): (capability set name not confirmed — verify). Expected pattern `UI-Circulation Settings <Resource>` is a naming convention only; Okapi permission names above are confirmed from `package.json`, their Eureka set names are not.
Permission edge cases: staff-slips `manage` alone must open `Print hold requests` without "Access Denied" (UICIRC-1356 fix, 12.0.2); `View circulation rules` and staff-slips perms must not show a 403 error modal (UICIRC-1225/1223, 11.0.1).

## ECS / Multi-Tenant Notes

- TLR pages differ when ECS requests are enabled (see Navigation). Consortium TLR + `Exclude tenants` only in ECS (UICIRC-1270).
- Policies and rules are per tenant; on member tenants a checkout must apply the member's rule — wrong/fallback policy after editing was a race condition, now fixed (CIRC-2522).
- ECS renewals use the loan policy applied at check-out (CIRC-2344).
- Shadow DCB locations selectable in the rules editor (UICIRC-1271).
- Pick slips for ECS requests print at the retrieval SP shown on the request (CIRC-2538).

## Existing automated coverage

- **Karate** (folio-integration-tests): `mod-circulation-storage/.../features/request-policies.feature` (allowed pickup SP validation), `.../circulation-settings.feature` (`printEventLogFeature`), `mod-circulation/.../features/loans.feature` (rule matching/priority, new rule via editor endpoint, overdue/lost policies), `loans-extended.feature` (anonymization with fee/fine rules), `requests.feature` (request policy type gating, pickup SP), `print-events*.feature`, `tlr-requests.feature`. No Karate for loan policy validation, fixed due date schedules, cancellation reasons, automatic-renewal fields.
- **Cypress** (stripes-testing `cypress/e2e/settings/circulation/`): loan/request/cancellation-reason/fixed due date CRUD, automatic renewals (4 specs = C1554460–C1554463), loan anonymization (C3616), other settings incl. Save state (C1385309), request policy pickup error (C410745), SP persistence (C410746), holds info icon (C411717), circulation rules edit, staff slips view permission/update, patron notices (6 specs). Fragments: `cypress/support/fragments/settings/circulation/*`, `fragments/circulation/{circulation-rules,loan-policy,fixedDueDateSchedules}.js`.

---

## Key Business Rules for Test Cases

1. When several rules match, the `priority:` line decides: highest criterium type first (default `t, s, c, b, a, m, g`), then number-of-criteria (locations count once), then first/last line; `fallback-policy` applies when nothing else matches. (github — mod-circulation `doc/circulationrules.md`; Karate `loans.feature`)
2. A policy in use by circulation rules (or a schedule used by loan policies) cannot be deleted — each type shows its own guard text. (github — en.json)
3. Loan profile `Fixed` requires a fixed due date schedule; `Rolling` requires an integer period > 0 and an interval; `Loan profile` and `Closed library due date management` are required while Loanable. (github — `Validation/loan-policy/loans.js`)
4. `Item limit` accepts 1–9999 or blank; over-limit checkout is blocked for the matched rule combination. (github — en.json `itemLimitInterval`; C9293)
5. Overdue fine policy: Maximum must be ≥ fine (also for recall fines); a fine amount > 0 is required once an interval is set. (github — en.json `finePolicy.validate.*`)
6. Lost item fee policy: processing fee > 0 only if declared-lost and/or aged-to-lost charging is `Yes`; replacement processing fee > 0 only if `Replacement allowed`; fractional fees such as 0.25 are valid. (github — en.json; UICIRC-1249)
7. Request policy gates Page/Hold/Recall per matched rule; every allowed pickup SP must be an existing pickup location, otherwise 422 `One or more Pickup locations are no longer available` (`INVALID_ALLOWED_SERVICE_POINT`). (Karate `request-policies.feature`; C410745)
8. A new request policy opens with defaults and must not inherit the previously edited policy's settings. (jira — UICIRC-1293)
9. `Allow title level requests` cannot be turned off while TLRs are open; Consortium TLR stays inactive until base TLR is checked and saved. (github — en.json; docs.folio.org)
10. If `Fail to create title level hold when request is blocked by circulation rule` is unchecked, TLR holds are always allowed. (github — en.json popover; docs.folio.org)
11. Wildcard lookup, audio alerts and session timeout are tenant-wide check-in/out settings, not per policy. (github — en.json `settings.checkout.*`)
12. Other settings: at least one patron identifier is required (`Please select to continue`); `User custom fields` needs ≥ 1 field (`Please make a selection`); Save is disabled when nothing changed or a warning shows. (jira — UICIRC-1316, UICIRC-1331; C1385309)
13. Loan anonymization runs hourly (≤ 50,000 loans per run); "after loan closes" intervals count from system return date; fee/fine loans follow the fee/fine rule from the last fee/fine close. (jira — CIRC-2459; Confluence "User-specific loan anonymization")
14. User-specific anonymization: `As early as possible` uses the earliest policy, `Never` is never anonymized, no preference or unknown user uses the library policy; with the feature off nothing changes. (jira — CIRC-2698, UICIRC-1361)
15. A rule line is valid only with one criteria expression and all five policy letters `l r n o i`; a missing type blocks Save and names the missing type. (cases — C651, C654)
16. Comments start with `#` or `/`; the `filter rules` box matches `#` section lines and treats special characters literally. (cases — C401724; github — circulationrules.md)
17. Closed library due date: "current"/"previous" options move the due date EARLIER, "next"/"beginning of next" move it LATER, "Keep" leaves it unchanged; long-term moves land on 11:59 PM; `Opening time offset` is added to "beginning of next open hours". (cases — C829891, C829892, C831956, C844247–C844249)
18. Short-term loans longer than 24 h with "Move to the end of the current service point hours" end at the close of the last open day before the closed day. (cases — C1621466; jira CIRC-2578)
19. Floating: check-in at a floating-enabled SP location moves Temporary + Effective location there; otherwise `In transit`; an open request overrides floating. (cases — C605980, C605981, C605982)
20. For use at location works only with `ENABLE_FOR_USE_AT_LOCATION=true`; needs a loan policy with `For use at location` + hold shelf expiration in days/weeks/months; circ actions `In use` / `Held`. (jira — CIRC-2457, UICIRC-1239; cases — C870003, C1307935)
21. For-use-at-location hold shelf expiration never goes past the loan due date; renewal extends it again. (jira — CIRC-2681, In Review)
22. Automatic renewals are available only while `Renewable` is checked; `Attempt first renewal` defaults to `On due date`; days-before = integer 1–100 (required when chosen); retries = integer 0–100 or blank (blank stored as 0). (jira — UICIRC-1351, UICIRC-1390; cases — C1554460–C1554463)
23. A loan policy without `automaticRenewal` is treated as disabled; the nightly job renews only eligible loans (policy on, inside window, renewal conditions pass, no hold/recall). (jira — CIRC-2687; CIRC-2584 not yet released)
24. Staff slip names are fixed and localized in the UI; the stored name must not be changed (printing matches by name); Preview always renders `{{staffSlip.Name}}` as `Transit`. (jira — UICIRC-1354, CIRC-2496)
25. In America/Vancouver, fixed due dates after 2026-11-01 must not shift by one hour; schedules saved before the fix keep the old offset until re-saved. (cases — C1594647–C1594650)
26. Unsaved form data in circulation settings must survive a token refresh (`/authn/refresh`). (cases — C916244; jira UICIRC-1287/1299)
27. View-only permission must not expose an active `Save` (TLR page) and view perms must not raise 403 error modals (rules, staff slips, print hold requests). (jira — UICIRC-1130, UICIRC-1225, UICIRC-1223, UICIRC-1356)
28. Shadow DCB locations are selectable only in the circulation rules editor, and a saved rule with them applies to DCB loans. (jira — UICIRC-1271)

---

## Authoring style (measured 2026-07-23)

Circulation Settings: **`Other` ~69%** / Func ~31%, median **~3 steps** (short), `User Journey` ~10%. Most cases are compact settings-config checks. Use `Other` for pure config/display; `Functional` where the setting drives a runtime effect verified downstream (e.g. closed-library due date). Heavy circulation-rule setup lives in Preconditions (rule example `m DVD : l <policy> r allow-all n send-no-notices o overdue-fine-policy i lost-item-fee-policy` — C844249); steps stay short. Preconditions list Okapi permission and Eureka capability side by side (table "Permission name for Okapi env." / "Capabilities/Sets for Eureka env." — C844249).

---

## Known Gaps / Items to Verify

- [ ] Release label: Jira names the next release "Umbrellaleaf 2027" in epics (FOLIO-4565, FOLIO-4577) while the module work is in progress (ui-circulation 13.0.0 IN_PROGRESS). Confirm the official label before tagging cases.
- [ ] User-specific loan anonymization (UICIRC-1361) is **In QA**: the story says section "Opt-in loan anonymization policy", en.json says `User-specific loan anonymization` (implementation deviated on purpose — Simon Lämmer comment). Assert en.json text; the "disable" warning dialog text is not in en.json yet.
- [ ] Automatic-renewal nightly job eligibility (CIRC-2584) is Blocked; FUAL recall cap of hold-shelf date (CIRC-2681 follow-up) not yet implemented.
- [ ] Incomplete-rule message `Must contain one of each policy type, missing type …` comes only from C654 (backend/editor text, not in ui-circulation en.json) — verify in env.
- [ ] Eureka capability-set names confirmed only for Loan-Policies manage and Other-Settings manage (Cypress `capabilitySets.js`) and staff slips manage / request-print-details view (UICIRC-1356; type prefix `Data` vs `Settings` and action casing not verified in UI); all others are marked (capability set name not confirmed — verify).
- [ ] `Treat closed loans with associated fee/fines differently` vs new fee/fine section header: both strings exist in en.json (`settings.loanHistory.treat`, `settings.loanHistory.feeFinePolicy`); which one is visible after UICIRC-1361 must be checked in env.
- [ ] docs.folio.org page (`Settings_circulation/settings_circulation.md`) has no Automatic renewals, For use at location or user-specific anonymization content yet — do not use it as source for those.
- [ ] Recall/hold alternate-period fields in Request management are listed from the original build; no new source checked their exact labels this round. Resolved 2026-11-26 for labels only: en.json `settings.requestManagement.*` confirms them (added `Alternate recall return interval for overdue loans` and `Alternate loan period at renewal…`); behaviour/validation of these fields still unchecked.

> History: build 2026-07-21 (browser); N≥10 audit 2026-07-22 (C651, C654, C196833, C196834, C401724, C605980–C605982, C829892, C844248, C870003, C869999, C1307935, C916267, C9293); spot-check 2026-07-23 (C831956); delta 2026-11-07 (automatic renewals, Vancouver TZ, CIRC-2496) — folded into the sections above on 2026-11-24. Resolved on 2026-11-24: long-term "end of the next open day" direction (C844249); circulation-rule priority (circulationrules.md; replaces "most specific wins").
