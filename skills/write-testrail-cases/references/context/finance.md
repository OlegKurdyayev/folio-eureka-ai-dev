# FOLIO Finance App - Business Logic Context

> Built: 2026-07-21 · Delta-refreshed: 2026-11-07 · Refreshed: 2026-11-24 · Release: Umbrellaleaf R2 2026 (ui-finance master package.json 9.1.0; mod-finance master 6.0.0-SNAPSHOT; last fixVersions seen: UIF 9.0.0 / MODFIN 5.2.0 / MODFISTO 8.9.0 = Trillium R1 2026) · Builder: build-app-context/manual — header normalized 2026-11-07, see area-map.md → Freshness

> Generated/refreshed by the build-app-context workflow on 2026-06-18.
> **Manual enrichment on 2026-07-21** (browser-based). TestRail cases C186156/C380517/C648502 supplied the Transaction detail-pane fields, Budget structure, Rollover page/dialogs and rollover rules. The public GitHub translation file `folio-org/ui-finance/translations/ui-finance/en_US.json` supplied every exact toast, error, confirm-dialog and create-form field label. "Exact UI Texts" is now complete for Finance; see Known Gaps for the (minor) remaining metadata staleness.
> Sources used in the 2026-06-18 run:
> - TestRail subtree scoped from group_id=454 (sections: 12, cases: 249 total)
> - Focus releases requested: Umbrellaleaf, Trillium, Sunflower, Ramsons, 2023 Poppy (0 exact-label matches in this subtree)
> - Release labels observed in this subtree include: Trillium (16), Sunflower (28), Ramsons (56), Poppy (as "R2 2023 Poppy", 24)
> - GitHub: folio-org/ui-finance, folio-org/mod-finance-storage (9 files sampled)
> - Jira: FOLIO component "Finance" (0 Done stories, 0 Done bugs returned in this run)
> This file combines stable Finance domain rules with run evidence for test-writing decisions.

---

## What is Finance

The Finance app manages library budget structures, tracks spending, and controls who can access financial records. It is tightly integrated with Orders (encumbrances) and Invoices (payments). All financial activity flows through Finance transactions.

---

## Record Hierarchy

```
Fiscal Year
  └── Ledger  (linked to one or more Fiscal Years)
        └── Fund  (belongs to exactly one Ledger)
              └── Budget  (one per Fund per Fiscal Year)
                    └── Transactions (Allocation, Encumbrance, Pending payment, Payment, Transfer, Credit)

Group  (cross-Ledger grouping of Funds - for reporting/summary only)
```

---

## Fiscal Year

### Required fields
- Name - e.g. "Fiscal Year 2026"
- Code - must start with alpha prefix followed by a 4-digit number (e.g. FY2026); the alpha prefix determines the fiscal year series and must be consistent for rollover (e.g. FYA2026 -> FYA2027)
- Period Begin Date (UTC)
- Period End Date (UTC)

### Key rules
- A single tenant can have multiple fiscal year series running in parallel (e.g. FYA for academic library, FYG for government library on different calendar years)
- The "current" fiscal year is determined by whether today's date falls within the Period Begin-End range, not by a manual flag
- A ledger can be associated with multiple fiscal years (for rollover history)
- Budget currency is locked at fiscal year creation time - set to the tenant primary currency when the fiscal year record is created; if tenant currency changes later, existing budgets remain in the original currency
- Optional: Acquisition units, Description

---

## Ledger

### Required fields
- Name
- Code - user-created, unique
- Fiscal year one - the first fiscal year for this ledger
- Status - Active / Frozen / Inactive

### Key fields
| Field | Notes |
|---|---|
| Enforce all budget encumbrance limits | Checked by default; if checked: FOLIO rejects encumbrances that exceed available budget |
| Enforce all budget expenditure limits | Checked by default; if checked: FOLIO rejects expenditures that exceed available budget |
| Acquisition units | Controls who can view/edit this ledger |
| Description | Optional |

### Ledger status values
- Active - ongoing; funds and budgets can be transacted against
- Frozen - paused; no new transactions allowed
- Inactive - no longer in use

### Ledger detail view
Shows: associated Funds with current Budget values, Groups, summary totals per fiscal year.

### Exporting ledger funds and budgets
- Ledger detail view -> Actions -> Export -> downloads CSV of all funds and budgets for the ledger

### Rollover
Triggered from ledger Actions -> Rollover:
1. Creates new budgets for the next FY (configurable allocation carry-over)
2. Creates new encumbrances for open orders in the next FY
3. Optionally closes current budgets
4. Does not automatically advance the "current" FY - determined by FY period dates
- Requires Finance rollover execute capability set

---

## Group

### Required fields
- Name
- Code
- Status - Active / Frozen / Inactive

### Key characteristics
- Cross-ledger grouping of funds for summary/reporting
- Does not restrict transactions - reporting only
- A fund can belong to multiple groups
- Optional: Acquisition units, Description

---

## Fund

### Required fields
- Name
- Code - must be unique; alphanumeric recommended
- Ledger - exactly one; cannot be changed after creation
- Status - Active / Frozen / Inactive

### Key fields
| Field | Notes |
|---|---|
| Currency | Pre-filled from the fiscal year currency; read-only after creation |
| Type | Optional category (e.g. Endowment, Restricted); used to group funds at rollover; funds without type grouped under "No fund type" at rollover |
| Acquisition units | Controls access; funds with restricted acquisition units are hidden from fund distribution dropdowns in Orders/Invoices for users outside those units |
| Group | Assign fund to one or more groups |
| Transfer from | Restrict which funds can transfer into this fund; blank = any fund allowed |
| Transfer to | Restrict which funds this fund can transfer to; blank = any fund allowed |
| External account number | Optional accounting system reference |
| Description | Optional |
| Donor information | Optional donor attribution |
| Locations | Optional location restrictions - limit fund use to specific library locations |
| Tags | Optional |

### Fund status
- Active - required for orders to be opened against this fund and for invoices to be paid
- Frozen - paused; no new transactions
- Inactive - no longer in use

---

## Budget

One budget per Fund per Fiscal Year.

### Required fields
- Name (auto-suggested from fund code + fiscal year code)
- Fiscal Year - selects which FY this budget covers

### Key fields
| Field | Notes |
|---|---|
| Allowable encumbrance (%) | Maximum percentage of allocation that can be encumbered; optional; if blank, no encumbrance limit beyond ledger setting |
| Allowable expenditure (%) | Maximum percentage of allocation that can be expended; optional |
| Status | Active / Planned / Closed / Frozen / Inactive (budget schema `budgetStatus` enum — acq-models `mod-finance/schemas/budget.json`; docs.folio.org Finance → Budget status; TestRail C663288 creates budgets in Inactive and Closed) |
| Expense classes | Add one or more expense classes to track spending by category within the budget |

### Budget financial buckets

| Bucket | Description | Formula |
|---|---|---|
| Allocated | Total money assigned to this budget | Sum of all Allocation transactions |
| Net transfers | Money moved in/out via Transfer transactions | Transfers in - Transfers out |
| Total funding | Allocated + Net transfers | Allocated + Net transfers |
| Encumbered | Reserved for open orders not yet invoiced | Sum of open Encumbrance transactions |
| Awaiting payment | Approved invoices not yet paid | Sum of Pending payment transactions |
| Expended | Paid invoices | Sum of Payment transactions |
| Unavailable | Encumbered + Awaiting payment + Expended | Sum of all committed/spent amounts |
| Over encumbered | Amount by which encumbrances exceed available | Only shows if ledger restrictions disabled |
| Over expended | Amount by which expenditures exceed available | Only shows if ledger restrictions disabled |
| Cash balance | Total funding - Expended | Remaining cash |
| Available | Total funding - Unavailable | Money not yet committed |

### Budget status values
Schema enum (acq-models `budget.json`): `Active`, `Frozen`, `Inactive`, `Planned`, `Closed`.
- Active - current; accepts transactions
- Planned - future budget; not yet active for transactions (FOLIO still allows a Planned budget in the current FY and an Active one in a future FY — MODFIN-407 comment)
- Closed - no longer accepting transactions; typically set during fiscal year rollover
- Frozen / Inactive - budget paused / no longer in use (docs.folio.org Finance → Budget fields)

### Expense classes
- Optional sub-categories within a budget (e.g. Monographs, Serials, Electronic)
- Configured in Settings -> Finance -> Expense classes
- Applied on Order lines and Invoice lines during fund distribution
- Each fund can have multiple expense classes on a budget

---

## Allocation Transactions

Allocation adds money to a budget. Three allocation operations:

### Increase allocation
- Budget -> Actions -> Increase allocation
- Enter amount -> Save -> Allocation transaction created
- Requires allocation create capability set + view/edit fund and budget capability sets

### Decrease allocation
- Budget -> Actions -> Decrease allocation
- Reduces the allocated amount
- Cannot decrease below what is already committed (unavailable)

### Move allocation (between funds)
- Moves money from one fund budget to another fund budget
- Budget -> Actions -> Move allocation
- Both funds must be in the same ledger (or have transfer restrictions satisfied)

### Batch allocations
- Multiple funds can be allocated simultaneously:
  - Via UI - select multiple funds -> Actions -> Batch allocate -> enter amounts per fund
  - Via **allocation worksheet (CSV)** - download the worksheet, edit it, upload it back
- Batch allocation jobs are logged; logs can be viewed and deleted (separate capability sets: `Data - UI-Finance Fund-Update-Logs - View` / `- Delete`)

#### Allocation worksheet (CSV) — download & upload (confirmed C648502/C648514/C651511 etc.; refs UIF-525, UIF-526, MODFIN-391, MODFIN-421)

- **Download:** from a **Ledger view** → `Actions > Download allocation worksheet (CSV)` → **"Select fiscal year"** modal (`Fiscal year` dropdown, **newest FY selected by default**, `Cancel` / `Confirm`). Confirm shows toast `Please wait while the worksheet is generated. Your download will start automatically` and downloads a CSV.
- **Worksheet columns:** Fund columns are always populated — `Fiscal year`, `Fund name`, `Fund code`, `Fund UUID`, `Fund status`; Budget columns — `Budget name`, `Budget UUID`, `Budget status`, `Budget initial allocation`, `Budget current allocation`, … — are **blank when the fund has no budget for that FY** (that's how a budget gets *created* on upload).
- **Upload:** `Actions > Upload allocation worksheet (CSV)` **creates or updates budgets in batch** for the worksheet's fiscal year — works for both the **Current** and a **Future** fiscal year (C648514 / C651511). A blank budget row → budget created; a filled row → budget updated.
- **Upload validations (each its own negative case):** rejects a file that is not `.csv`, a worksheet with a **missing column**, a CSV that **doesn't correspond** to the generated worksheet, rows whose **fiscal years differ**, a **fund belonging to another Ledger**, and **invalid fund/budget status** values. Batch-allocation access is also capability-gated (a user without the capability can't reach the worksheet actions).

---

## Transfer Transactions

Transfers move money between two fund budgets.

### Creating a transfer
- From budget: Actions -> Transfer
- Specify: From fund (source), To fund (target), Amount, Fiscal year
- Transfer restrictions on funds (Transfer from/to fields) are enforced
- Requires transfer create capability set

### Transfer restrictions
- If a fund has Transfer from restrictions: only listed funds can transfer into it
- If a fund has Transfer to restrictions: only listed funds this fund can transfer to
- Blank fields = no restrictions

---

## Encumbrances

Created automatically by Orders when a PO is opened. Not manually creatable in Finance UI.

### Manually releasing an encumbrance
- Budget transaction log -> select encumbrance -> Release encumbrance
- Requires encumbrance manual release capability set
- Use case: vendor cancelled; free funds before invoice arrives

### Unreleasing an encumbrance
- Budget transaction log -> select released encumbrance -> Unrelease encumbrance
- Requires encumbrance unrelease capability set
- Use case: encumbrance released by mistake during invoice approval

---

## Transactions

All financial activity is recorded as immutable transactions. Transactions cannot be edited or deleted.

| Transaction type | Created by | Budget impact |
|---|---|---|
| Allocation | Finance app (manual) or Rollover | Allocated increases |
| Transfer | Finance app (manual) | Source Available decreases, Target Available increases |
| Encumbrance | Orders (on PO Open) | Encumbered increases, Available decreases |
| Pending payment | Invoices (on Approve) | Awaiting payment increases, Encumbered decreases |
| Payment | Invoices (on Pay) | Expended increases, Awaiting payment decreases |
| Credit | Invoices (credit invoice) | Available increases |
| Rollover transfer | Fiscal year rollover | Carries allocation/encumbrance to new FY |

### Viewing transactions
- Budget detail view -> Transactions accordion -> filter by type, date, amount
- Typical columns: Transaction date, Type, Amount, Source (Order/Invoice/Manual), Tags

---

## Acquisition Units

Acquisition units gate access to Finance records (same behavior as in Orders and Invoices).

### How they work
- Each Finance record (Fiscal Year, Ledger, Fund, Group) can have one or more Acquisition units assigned
- User must be a member of at least one assigned unit to view/edit the record
- Users with bypass acquisition unit capability can access all records
- Hidden from fund distribution: funds with acquisition unit restrictions are filtered out of fund selection dropdowns in Orders/Invoices for users outside those units

### Assign on create vs manage on edit
- Assign acquisition units to new record (Procedural execute) - only allows assigning units on create
- Manage acquisition units (Data manage) - allows changing unit assignment when editing existing records

---

## Recalculate Budget Totals

In case of data inconsistency (e.g. transactions deleted administratively):
- Budget -> Actions -> Recalculate budget totals
- Recomputes all bucket values from actual transactions
- Requires recalculation execute capability set

---

## Deleting Records

| Record | Can be deleted when... |
|---|---|
| Budget | Status is Planned or Closed; no transactions |
| Fund | No active budget with transactions; not referenced by open orders or unpaid invoices |
| Ledger | No active funds |
| Group | Always deletable (no transactional dependency) |
| Fiscal Year | Not referenced by any ledger |

---

## Rollover Fiscal Year

Triggered from a Ledger -> Actions -> Rollover.

### Rollover configuration options
- Rollover type: whether to roll over encumbrances (open orders) into new FY
- Allocation strategy: carry over full amount / carry over remaining amount / no carry over
- Close budgets: optionally close current FY budgets after rollover
- Fund types: rollover can be configured per fund type

### What rollover does
1. Creates new Budget records for each fund in the ledger for the new fiscal year
2. Creates new Encumbrance transactions in new budgets for each open PO with encumbrances in current year
3. Optionally closes current year budgets (status = Closed)
4. Generates a rollover log with results

### Rollover log
- Viewable after rollover: shows successes, errors, funds processed
- Accessible from: Ledger -> Rollover logs

---

## Exact UI Texts

> Use these verbatim in expected results. Never paraphrase.
> ✅ **Fully seeded on 2026-07-21.** Modal/transaction-pane/budget/rollover fields were mined from TestRail cases (C186156, C380517, C648502); all toast, error, dialog and form-field strings below were pulled verbatim from the public GitHub translation file `folio-org/ui-finance` → `translations/ui-finance/en_US.json`. These are the exact rendered strings — safe to assert directly. `{placeholders}` interpolate at runtime.

### Transaction detail pane — fields (the core Finance verification surface)

When a step opens a transaction (Encumbrance, Payment, Pending payment, Credit, Allocation, Transfer) from a budget's `View transactions`, the detail pane shows these fields — verify column-by-column with exact values [confirmed — C186156]:
`Transaction date`, `Fiscal year`, `Amount`, `Source` (e.g. the PO line, hyperlinked), `Type`, `From` (source fund), `To` (target fund), `Expense class`, `Tags`, `Initial encumbrance`, `Awaiting payment`, `Expended`, `Status`, `Description`.
Encumbrance `Status` values: `Unreleased`, `Released`, `Pending`.

### Budget page structure

- Fund pane accordions: `Current budget`, `Planned budget` (a rolled-over budget appears here named `<Fund code> - <FY code>`, e.g. "Fund A - FY #2"). [confirmed — C186156]
- Budget page accordions: `Budget summary` (Funding information table: Allocated, Net transfers, Total funding, Encumbered, Awaiting payment, Expended, Unavailable, Over encumbered/expended, Cash balance, Available), `Budget information` (contains `Status` field and the `View transactions` link), `Expense classes`.
- Budget `Status` values: `Active`, `Planned`, `Closed` seen in rollover cases (C186156); the schema also has `Frozen`, `Inactive`.

### Modal / Dialog Fields & Titles

| Modal | Fields / elements | Source |
|---|---|---|
| Move allocation | `From` dropdown, `To` dropdown (pre-filled with the fund whose budget you opened), `Amount*` field, `Tags` dropdown, `Description` field, a **`Swap` button/icon that instantly swaps the `From`/`To` selections** (re-evaluates the negative-allocation validation live — swapping can flip `Confirm` from disabled to enabled or vice versa), `Cancel` button, `Confirm` button (blue, highlighted) | [confirmed — C380517, C825298] |
| Increase allocation / Decrease allocation | Opened from Budget → Actions; amount entry + Confirm | [from cases] |
| Transfer | `From` / `To` budget fields, Amount, Confirm | [from cases — C6650] |
| Unpaid invoices (during rollover) | message `FOLIO has found invoices that are not yet paid or canceled. If you are sure you want to continue with rollover click continue.` + `Continue` button | [confirmed — C186156] |
| Fiscal year rollover (confirm) | `Confirm` button; on success a toast + `Close & view ledger details` button appear | [confirmed — C186156] |

### Rollover page — fields (Ledger → Actions → Rollover)

Top-level: `Fiscal year` dropdown; `Enforce all budget encumbrance limits during rollover` checkbox (labelled `Enforce all budget encumbrance limits` before Ramsons); `Enforce all budget expenditure limits during rollover` (removed since Trillium); `Close all current budgets` checkbox. [confirmed — C186156]
`Rollover budgets` accordion: `Rollover allocation` checkbox, `Adjust allocation, %` field, `Rollover budget value` dropdown (options `None`, `Available`, `Cash balance` — en.json `ledger.rollover.rolloverBudgetValue.*`), `Rollover value as` dropdown (`Transfer` / `Allocation` — `ledger.rollover.addAvailableTo.*`), `Set allowances` checkbox, `Allowed encumbrance, %`, `Allowed expenditure, %`.
`Rollover encumbrances` accordion: `Ongoing encumbrances` toggle, `Ongoing-subscription encumbrances` toggle, `One-time encumbrances` toggle, each with a `Based on` dropdown (options `Expended`, `Initial encumbrance`, `Remaining` — en.json `ledger.rollover.basedOn.*`; C375267/C400643 cover `Remaining`) and an `Increase by, %` field.

### Validation / Error Messages (inline)

| Condition | Message text | Behaviour | Source |
|---|---|---|---|
| Move/allocate amount would drive total allocation negative | `Total allocation cannot be less than zero` | Red message below the `Amount` field; `Confirm` becomes **disabled** (no error toast — validation is inline, the action cannot be submitted) | [confirmed — C380517] |

> Note the interaction model: insufficient-funds is caught by **inline validation that disables Confirm**, NOT by an error toast after submitting. Do not write a "click Confirm → error toast" flow for this scenario.

### Button / Action Labels (key actions)

Actions, Increase allocation, Decrease allocation, Move allocation, Transfer, Confirm, Cancel, Continue, View transactions, Rollover, `Close & view ledger details`, Export (Ledger), Save & close. [from cases]

### Toast Messages

> All rows below are **[confirmed — GitHub `ui-finance/translations/ui-finance/en_US.json`, pulled 2026-07-21]**. `{placeholders}` are interpolated at runtime; `<b>` marks bold segments in the rendered toast.

| Event | Toast text |
|---|---|
| Increase allocation | `{amount} was successfully allocated to the budget <b>{budgetName}</b>.` |
| Decrease allocation | `Budget <b>{budgetName}</b> allocation was successfully decreased by {amount}.` |
| Move allocation | `{amount} was successfully allocated to the budget <b>{budgetName}</b>.` |
| Transfer | `{amount} was successfully transferred to the budget <b>{budgetName}</b>.` |
| Budget saved | `Budget <b>{name}</b> has been saved` |
| Budget created | `Budget <b>{name}</b> successfully <b>created</b> for fund <b>{fundName}</b>` |
| Fund saved | `Fund has been saved` |
| Fund deleted | `Fund has been deleted` |
| Ledger saved | `Ledger has been saved` |
| Ledger deleted | `Ledger has been deleted` |
| Group saved | `Group has been saved` |
| Group deleted | `Group has been deleted` |
| Fund added to group | `Fund(s) have been added to group` |
| Fiscal year saved | `Fiscal year has been saved` |
| Fiscal year deleted | `Fiscal year has been deleted` |
| Release encumbrance | `Encumbrance was released` |
| Unrelease encumbrance | `Encumbrance was unreleased` |
| Recalculate budget totals | `Budget totals have been updated based on current budget transactions` |
| Rollover complete | `Rollover is complete` (header `Rollover {fromYearCode} to {toYearCode}` on the Ledger name pane) |
| Batch allocations updated | `Allocations have been updated successfully.` |
| Batch budgets created | `Budgets have been created successfully.` |
| Export budget CSV started | `Export of <b>{name}</b> data has started` |
| Export budget CSV success | `<b>{name}</b> data was successfully exported to CSV` |
| Budget creation failed | `Budget has not been created` [from source — `finance/states.js`, 2026-08-06] |
| Rollover test started | `{name} rollover test has started successfully` [from source — `finance/states.js`, 2026-08-06] |
| Rollover budget results export started | `Export of budget results has been started` [from source — `finance/states.js`, 2026-08-06] |

### Error / Validation Messages (confirmed — GitHub translations, 2026-07-21)

| Condition | Message text |
|---|---|
| Allocation/transfer would make total allocation negative | `Total allocation cannot be less than zero` (inline; disables Confirm) |
| Allocate more than source has | `{amount} was not successfully allocated to the budget <b>{toBudgetName}</b> because it exceeds the total allocation amount of <b>{fromBudgetName}</b>.` |
| Allocate more than source has, restrictions active | `{amount} was not successfully allocated to the budget <b>{toBudgetName}</b> because it exceeds the total allocation amount of <b>{fromBudgetName}</b> and ledger fund restrictions are active.` |
| Transfer exceeds expenditure limit (restrictions active) | `$X.XX could not be transferred to the budget {firstBudgetName} because it exceeds the allowable expenditure limit for {secondBudgetName} and ledger fund restrictions are active.` [from source — `finance/states.js`, 2026-08-06; `X.XX` = the formatted dollar amount] |
| Transfer more than Available | `Transfer was not successful. There is not enough money Available in the budget to complete this Transfer.` |
| Same-budget transfer/allocation | `Transfers and allocations can not be made from one budget to that same budget` |
| Transfer restriction (allow transfer to/from) blocks it | `This transaction cannot be made between these two Funds as the "allow transfer to or from" Fund setting on one or more of the Funds is too restrictive.` |
| Fund name duplicate on ledger | `This Fund name is already in use on the selected Ledger.` |
| Fund code duplicate | `This Fund code is already in use.` |
| Ledger name/code duplicate | `This Ledger name is already in use.` / `This Ledger code is already in use.` |
| Fiscal year code duplicate | `This Fiscal year code is already in use` |
| Fiscal year code format | `Fiscal year Code format must be an Alpha followed by a four digit numeric. Eg. ExampleFY2020` |
| Fiscal year period invalid | `Fiscal year has not been saved. Fiscal year period end date is earlier than start date` |
| Delete budget with transactions | `Budget can not be deleted, because it has transactions` |
| Delete budget with expense classes | `Unable to delete budget. You can not delete a budget that has been assigned one or more expense classes.` |
| Negative available (confirm modal) | heading `Negative available amount`; message `Completing this transaction will result in <b>{budgetName}</b> having a negative available amount. Are you sure you would like to complete this transaction?` |

### Confirm / Delete dialogs (confirmed — GitHub translations)

| Dialog | Heading / message |
|---|---|
| Delete fund | `Delete fund` / `Are you sure you want to delete fund?` |
| Delete ledger | `Delete ledger` / `Are you sure you want to delete ledger?` |
| Delete group | `Delete group` / `Are you sure you want to delete group?` |
| Delete budget | `Delete budget` / `Are you sure you want to delete budget?` |
| Delete fiscal year | `Delete fiscal year` / `Are you sure you want to delete Fiscal year` |
| Release encumbrance | `Release encumbrance` / `Are you sure you want to release this encumbrance? Any remaining amount will be added back to the budget.` |
| Unrelease encumbrance | `Unrelease encumbrance` / `Are you sure you want to unrelease this encumbrance? Any remaining amount will be encumbered against the budget.` |
| Rollover confirm | `Fiscal year rollover` / `Ledger <b>{ledgerName}</b> for <b>{currentFYCode}</b> will be rolled over to <b>{chosenFYCode}</b>. Please confirm that you have completed the necessary details and are ready to proceed with rollover.` [en.json `ledger.rollover.confirm.message`, 2026-11-24; the older `en_US.json` wording ending "may take several minutes ... cannot be undone" is superseded] |
| Save fund (duplicate name) | `Save fund` / `Warning: You are about to add a fund with the same name as an existing fund. Would you like to complete this action?` |

### Fiscal Year detail view — sub-table columns (confirmed from `financeDetails.js`, 2026-08-06)

The Fiscal Year detail view contains three sections (ids `ledger`, `group`, `fund`) each with a table. Common columns: `Name`, `Code`, `Allocated`, `Unavailable`, `Available`. Empty table text: `The list contains no items`.

Financial summary section (id `financial-summary`) text labels: `Cash balance: {value}` and `Available balance: {value}` (rendered as inline HTML, not key-value pairs).

### Budget Expense classes table columns (confirmed from `financeDetails.js`, 2026-08-06)

The `Expense classes` accordion inside a budget shows a table with these columns: `Expense class`, `Encumbered`, `Awaiting payment`, `Expended`, `Percent of total expended`, `Status`. An `Unassigned` row (with info icon tooltip: `applies to transactions completed prior to the inclusion of expense classes on the budget, if there are any that exist`) appears for transactions made before expense classes were applied.

### Create-form titles & field labels (confirmed — GitHub translations)

- Form titles: `Create fund`, `Create ledger`, `Create fiscal year`, `Create group` (edit variants: `Edit fund`/`Edit ledger`/`Edit fiscal year`/`Edit group`).
- **Fund** fields (Fund information): `Name`, `Code`, `Ledger`, `Status`, `Currency`, `Type`, `Group`, `Acquisition units`, `Transfer from`, `Transfer to`, `External account`, `Description`, `Donor information`, `Restrict use by location`, `Locations`. Statuses: `Active`, `Frozen`, `Inactive`.
- **Ledger** fields (Ledger information): `Name`, `Code`, `Status`, `Fiscal year one`, `Description`. Also `Current fiscal year`. Statuses: `Active`, `Frozen`, `Inactive`.
- **Fiscal year** fields: `Name`, `Code`, `Period Begin Date (UTC)`, `Period End Date (UTC)`, `Description`.
- **Group** fields (Group information): `Name`, `Code`, `Fiscal year`, `Status`, `Description`. Statuses: `Active`, `Frozen`, `Inactive`.

---

## Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View fiscal years | Data - UI-Finance Fiscal-Year - View |
| Create/edit fiscal years | Data - UI-Finance Fiscal-Year - Create/Edit |
| Delete fiscal years | Data - UI-Finance Fiscal-Year - Delete |
| View ledgers | Data - UI-Finance Ledger - View |
| Create/edit ledgers | Data - UI-Finance Ledger - Create/Edit |
| Delete ledgers | Data - UI-Finance Ledger - Delete |
| View groups | Data - UI-Finance Group - View |
| Create/edit groups | Data - UI-Finance Group - Create/Edit |
| Delete groups | Data - UI-Finance Group - Delete |
| View funds and budgets | Data - UI-Finance Fund-Budget - View |
| Create/edit funds and budgets | Data - UI-Finance Fund-Budget - Create/Edit |
| Delete funds and budgets | Data - UI-Finance Fund-Budget - Delete |
| Create allocations | Data - UI-Finance Allocations - Create |
| Create transfers | Data - UI-Finance Transfers - Create |
| Execute fiscal year rollover | Procedural - UI-Finance Ledger Rollover - Execute |
| Manually release encumbrance | Procedural - UI-Finance Encumbrance Release-Manually - Execute |
| Unrelease encumbrance | Procedural - UI-Finance Encumbrance Unrelease - Execute |
| Recalculate budget totals | Procedural - UI-Finance Fund-Budget Recalculate-Totals - Execute |
| Export finance records | Procedural - UI-Finance - Execute (UI permission `ui-finance.exportCSV`) |
| Assign acquisition units to new record | Procedural - UI-Finance Acq Unit Assignment - Execute (UI permission `ui-finance.finance.acquisitions_units.assign.execute`) |
| Manage acquisition units on existing record | Data - UI-Finance Acq Unit Assignment - Manage |
| View batch allocation logs | Data - UI-Finance Fund-Update-Logs - View |
| Delete batch allocation logs | Data - UI-Finance Fund-Update-Logs - Delete |

---

## Key Business Rules for Test Cases

1. Budget values are read-only and derived from transactions; they cannot be directly edited
2. Budget currency is locked at fiscal year creation; tenant currency changes do not retroactively alter existing budgets
3. Fund belongs to exactly one ledger and cannot be reassigned after creation
4. Ledger enforce encumbrance/expenditure limits are checked by default; if enabled, overdraw transactions are blocked
5. Acquisition units gate access and hide restricted funds from Orders/Invoices distribution dropdowns
6. Fund must be Active for orders to open and invoices to pay
7. Groups are reporting-only and do not affect transaction processing
8. Current fiscal year is date-driven, and rollover does not manually set current FY
9. Transactions are immutable; corrections require recalculate or compensating transactions
10. One budget per fund per fiscal year
11. Closed budgets block new transactions
12. Fund type drives rollover grouping; untyped funds fall under "No fund type"
13. Budget allowable encumbrance/expenditure percentages can impose additional limits
14. Transfer restrictions are directional and enforced on both incoming and outgoing paths
15. Assign vs Manage acquisition unit capability sets are distinct and independently required
16. Rollover requires a dedicated procedural capability set
17. Release and Unrelease encumbrance are separate procedural capability sets
18. Batch allocation log deletion capability set is separate from viewing logs
19. Fund code uniqueness is global across ledgers
20. Expense classes are optional but transaction-visible and used for spending analysis
21. **Rollover creates a `Planned budget` for each fund in the new FY** and (if `Close all current budgets` is set) moves the source budgets to `Closed`; encumbrances roll per the Rollover encumbrances toggles. Rolled encumbrances land as `Unreleased` in the new-FY budget; the closed-FY copy reflects the pre-rollover end state. (cases — C186156)
22. **Rollover is blocked-with-warning by unpaid invoices** — the `Unpaid invoices` dialog appears and requires explicit `Continue` before the rollover confirm. (cases — C186156)
23. **A fiscal year series is identified by its alpha code prefix** (e.g. `FYA2025` → alpha `FYA`); rollover requires the next FY to share the same alpha. (cases — C186156)
24. **A ledger can only be rolled over once from a given source fiscal year.** After Ledger rolls FY-A → FY-B, attempting a second rollover from the ledger's (now-current) FY-B to a further FY-C does not error on FY-C being invalid — it errors because the ledger *already has a completed rollover on record*: `"{ledgerName} was already rolled over from the {fromYearCode} fiscal year to the {toYearCode} fiscal year"`, and the second rollover does not run. Test this as its own scenario (rollover once → succeed; immediately rollover again → blocked with this exact message), not just "rollover succeeds." (cases — C360956)
25. **"Move allocation" and "Transfer" enforce negative-available differently** — Move allocation is a **hard block**: if the amount would drive total allocation negative, `Confirm` is disabled with the inline message `Total allocation cannot be less than zero` and the action cannot be submitted at all. Transfer, by contrast, **allows** a resulting negative available balance but requires an extra confirmation step: clicking `Confirm` first opens a `Negative available amount` dialog (`Completing this transfer will result in <budgetName> having a negative available amount. Are you sure you would like to complete this transaction?`); only a second `Confirm` on that dialog commits it. Don't write these two actions as having the same negative-balance behavior — one is blocked outright, the other is allowed-with-warning. (cases — C374166, C374183)
26. **Remaining allowed encumbrance is computed, not just compared to a flat cap**: `remaining allowed encumbrances = (allocated + netTransfers) × allowableEncumbered% − (encumbered + awaitingPayment + expended)`. When a PO's requested encumbrance would exceed this remaining figure (with "Enforce all budget encumbrance limits" active on the Ledger), Opening the order fails with the toast `One or more fund distributions on this order can not be encumbered, because there is not enough money in [{Fund name}].` — order stays `Pending`, the POL's Current encumbrance shows `-`, and no encumbrance transaction is created for that order at all (verify via Fund > Actions > View transactions for current budget). This is the Orders-side symptom of a Finance-side limit; when writing an Orders test for this, cross-check the encumbrance math against this formula rather than guessing a round-number threshold. (cases — C449364)
27. **"Available" (and similarly negative budget figures) render in parentheses, not with a minus sign**, since Ramsons — e.g. `($500.00)` for a budget that is $500 over. Pre-Ramsons cases/screenshots show a leading `-` instead; don't assert the old format against a current-release environment. (cases — C374183, C496145)
28. **A Fund's "Restrict use by location" checkbox blocks Save & close on its own, with no separate confirm step.** Checking it expands a `Locations` accordion showing `Locations must be assigned` while empty; attempting to Save & close with the checkbox checked and zero locations added leaves the user on the Create/Edit page with that warning still showing — it does not throw a toast or navigate away. Unchecking the box removes the requirement immediately (accordion disappears, Save & close proceeds normally). Test both the blocked-save path and the unchecked-passthrough path, not just "restricted funds require locations" as a single assertion. (cases — C423530)
29. **Rollover based on "Expended" promotes the Planned budget to Active and folds the old Current budget's allocation into it** — after rollover, the old FY's budget becomes `Closed`; the new FY's Planned budget becomes `Active`, with `Initial allocation` = the Planned budget's own pre-set allocation, `Increase in allocation` = the old Current budget's allocated amount, and `Total allocated` = the sum of both. Additionally, any **paid-invoice amount that exceeded the old POL's fund-distribution value** rolls forward as a new `Unreleased` encumbrance in the new fiscal year, sourced to the same PO number, with `Initial encumbrance` = that paid amount and `Awaiting payment`/`Expended` reset to $0.00. Don't test "Planned → Active" and "encumbrance carries the overage" as unrelated facts; they're two sides of the same rollover-with-Expended-basis behavior. (cases — C359186)
30. **With `Rollover allocation` OFF and `Rollover budget value` = `None`, the new FY's Planned budget still gets created and set `Active` — just entirely zeroed out**: `Initial allocation`, `Increase/Decrease in allocation`, `Total allocated`, `Net transfers`, `Unavailable`, and `Available` all read `$0.00`/`0`. This refines Rule 21 — "a Planned budget is created for each fund" doesn't by itself imply it carries any money forward; the specific combination of toggles determines whether it's a real starting balance or an empty shell. (cases — C376609)
31. **Budget `status` has five schema values** (`Active`, `Frozen`, `Inactive`, `Planned`, `Closed`), not three; budgets can be created in Inactive/Closed status, but not with negative allocation or negative allowable percentages. (acq-models `budget.json`; UIF-573, MODFIN-405; C663288)
32. **Batch allocation (worksheet / Batch edit) is limited to 1000 funds per ledger** and is available from the Ledger Actions → Allocation tools only (not Fiscal year, Fund; removed from Group). (UIF-647, UIF-525, UIF-598; C648520)
33. **New budget created by batch allocation gets status `Active` in the current FY and `Planned` in a future FY** unless the user sets a valid status. (MODFISTO-517, UIF-596; C651440, C651433)
34. **Increasing allocation on an Inactive/Frozen fund via batch allocation makes the fund Active**; non-increasing changes leave the fund status alone. (MODFIN-407; C651420)
35. **`0` is a valid allocation (budget created with no allocation transaction); a resulting total allocation below zero is rejected** on Increase/Decrease/Move allocation, Batch edit (`New total allocation cannot be negative.`) and budget create. (UIF-576, UIF-621, UIF-573; C825298, C374166)
36. **If any budget in a batch allocation is not Active/Planned the whole update fails** with `Allocations have not been updated because one or more budgets are not active or planned.` (UIF-651; en.json `errors.failedToCreateAllocations`)
37. **Worksheet upload is rejected for**: non-CSV file, missing required headers, unparsable file, empty file, unresolvable or inconsistent fiscal year, duplicate (FY, budget, fund) rows, unknown Budget/Fund UUID, fund of another ledger, invalid status text; each has its own en.json message. (en.json `batchAllocations.uploadWorksheet.validation.error.*`; MODFIN-403; C651417, C651435, C651423, C651422)
38. **Batch allocation logs are restricted by acquisition units** of the ledger; Group-based log cases are DEFERRED. (MODFIN-416, UIF-588; C688780, C663299)
39. **Batch Edit `Save & close` disabled after changes made post-Recalculate; `Unsaved changes` modal on cancel.** (UIF-591, UIF-652; C1292042)
40. **Only one row per unique budget in Batch edit and worksheet after rollover.** (MODFIN-412, UIF-587; C656288)
41. **`PUT /finance/finance-data` supports preview mode with no DB commit and no log entry.** (MODFIN-382; Karate `finance-data.feature`)
42. **Rollover with only some encumbrance order types selected must leave unselected types with the "none selected" outcome (Released, $0.00, PO line stays on current-FY encumbrance).** (MODFISTO-559, MODFISTO-560; C1347127)
43. **Rollover `Based on` supports `Expended`, `Initial encumbrance`, `Remaining`; `Rollover budget value` supports `None`, `Available`, `Cash balance`; `Rollover value as` supports `Transfer`/`Allocation`.** (en.json `ledger.rollover.*`; C376601–C376611, C400643)
44. **Rollover budget % adjustments must round to currency precision.** (MODFIN-439; C831977)
45. **Rolled-over `Rollover transfer` amounts are counted in `Net transfers` after Recalculate.** (MODFIN-394; C627392)
46. **After a multi-ledger PO line rollover, budget Encumbered must be correct for every ledger without a manual recalculation.** (MODFISTO-549; C397990)
47. **Rollover cannot be re-run for the same ledger/FY pair; test rollover is blocked after the common rollover.** (en.json `ledger.rollover.error.conflict`; C402327, C360956)
48. **Rollover results/errors logs are not limited to 10 records and export works for funds without a budget.** (UIF-455, UIF-607; C380629, C770425)
49. **Rollover logs error CSV does not yet fill fund code / PO line number.** (UIF-624; C399078 marked NOT implemented)
50. **`Enforce all budget expenditure limits during rollover` no longer exists** in the rollover form or in log settings. (UIF-611)
51. **Unrelease of an encumbrance is blocked when it has pending payment, credit or expended transactions.** (MODFISTO-542 comment)
52. **Cancelling approved invoices / paid credit invoices must give correct encumbered amount** (cap to zero only at the end). (MODFISTO-537, MODFISTO-542, MODFISTO-512)
53. **Budget summaries show credited totals separately from expended** (FY, Ledger, Group, Budget; CSV export). (UIF-494..498, MODFIN-371, MODFISTO-481; Cypress `export-budget-credited-columns.cy.js`)
54. **Expense class tables include an `Unassigned` first row for transactions without an expense class** (fund, budget, group). (UIF-617, MODFIN-433)
55. **Fiscal year currency is stored on the FY and must not change when the FY is edited or when the system currency changes.** (MODFIN-386, UIF-538; Cypress `currency-consistency-for-fiscal-year-and-budget-after-changing-currency-settings.cy.js`)
56. **FY Period End Date means end of the selected day (23:59 UTC).** (UIF-666)
57. **When the current FY of a ledger lapses, UI shows `No current fiscal year found.` (and `No planned fiscal year found.` for planned lookups); no rollover-complete toast.** (UIF-645, MODFIN-445, UIF-539; C514909)
58. **Fund restricted by location in ECS shows only locations from the user's affiliations.** (UIF-561, UIF-606; C466311, C468164)
59. **Exchange rate source setting needs `Settings (Finance): Manage external exchange rate source`; ECB is the default; Key required type makes API key mandatory.** (UIF-583, UIF-605; docs Settings_finance; C736705, C736706)
60. **Browse tab is controlled by Settings → Finance → Navigation `Enable browse tab`.** (UIF-544; en.json `settings.navigation.*`; C788678 draft)
61. **Optimistic locking applies to all Finance entities; conflict shows a banner/message, not a generic error.** (MODFISTO-303, MODFISTO-513, UIF-565; C353947–C353961)
62. **Fund-type and expense-class deletion is blocked when in use.** (docs Settings_finance; C367941, C15857)
63. **Transfer/Allocation error toasts depend on the cause**: not enough money, ledger restrictions, no budget for fund, restrictive "allow transfer to or from", same-budget; all keyed in en.json `transaction.transfer.*` / `transaction.allocation.*`. (UIF-543, UIF-604, UIF-619; C380517, C825296)
64. **`Move allocation`/`Transfer` `Swap` re-evaluates validation live.** (UIF-524; C808443, C825298)

---

## Authoring style (measured 2026-07-23)

Finance is strongly **`Functional` (~94%)**, median ~7 steps, `User Journey` ~2% (`No`). Cases live and die on **exact monetary values**: preconditions establish the full hierarchy (Fiscal year with today in its period, active Ledger, Fund with a current Budget and specific `Allocated`/`Encumbered`/`Available`) and steps verify budget buckets **absolutely, bucket-by-bucket** (`Initial allocation`, `Increase in allocation`, `Total allocated`, `Net transfers`, `Encumbered`, `Awaiting payment`, `Expended`, `Unavailable`, `Available`), plus the underlying Transaction rows (Type/Source/Amount/Status). Multi-fiscal-year features (rollover, batch-allocation worksheet, future-FY budgets) require the previous+current(+future) FY with an identical letter part as preconditions. `refs` frequently span sibling FE/BE tickets (e.g. `UIF-525, UIF-526, MODFIN-391, MODFIN-421`). Match this: concrete amounts, `Functional` type, column-level budget/transaction verification.

---

## Status Values and Transaction Types (schema-sourced, 2026-11-24)

| Record | Values | Source |
|---|---|---|
| Fund `fundStatus` | `Active`, `Frozen`, `Inactive` | acq-models `mod-finance/schemas/fund.json` |
| Ledger `ledgerStatus` | `Active`, `Inactive`, `Frozen` | `ledger.json` |
| Budget `budgetStatus` | `Active`, `Frozen`, `Inactive`, `Planned`, `Closed` | `budget.json` |
| Transaction `transactionType` | `Allocation`, `Credit`, `Encumbrance`, `Payment`, `Pending payment`, `Rollover transfer`, `Transfer` | `transaction.json` |
| Transaction `source` | `User`, `PoLine`, `Invoice` | `transaction.json` |
| Batch allocation log status | `Completed`, `Error`, `In Progress` | en.json `allocation.batch.logs.columns.status.*` |
| Rollover log status | `Successful`, `Failed`, `In progress`, `Not started` | en.json `ledger.rollover.status.*` |

Required on create: Fund — code, fundStatus, ledgerId, name; Ledger — name, code, fiscalYearOneId, ledgerStatus, restrictEncumbrance, restrictExpenditures; Budget — budgetStatus, name, fundId, fiscalYearId (same schema files). `Rollover transfer` is a transaction type of its own (also filterable via the Transactions `Encumbrance status` filter for encumbrances — UIF-535).

---

## API Surface (mod-finance / mod-finance-storage descriptors, master 2026-11-24)

Source: `descriptors/ModuleDescriptor-template.json` of mod-finance (provides `finance.*`) and mod-finance-storage (`finance-storage.*`). `permissionsRequired` follow the pattern `finance.<resource>.<collection|item>.<get|post|put|delete>`.

| Endpoint (mod-finance) | permissionsRequired |
|---|---|
| `GET/POST /finance/fiscal-years`, `GET/PUT/DELETE /finance/fiscal-years/{id}` | `finance.fiscal-years.collection.get` / `.item.post` / `.item.get` / `.item.put` / `.item.delete` |
| `GET/POST /finance/ledgers`, `GET/PUT/DELETE /finance/ledgers/{id}` | `finance.ledgers.*` |
| `GET /finance/ledgers/{id}/current-fiscal-year`, `/planned-fiscal-year` | `finance.ledgers.current-fiscal-year.item.get`, `finance.ledgers.planned-fiscal-year.item.get` |
| `GET/POST /finance/funds`, `GET/PUT/DELETE /finance/funds/{id}`, `GET /finance/funds/{id}/budget`, `/expense-classes` | `finance.funds.*`, `finance.funds.budget.item.get`, `finance.funds.expense-classes.collection.get` |
| `GET/POST /finance/budgets`, `GET/PUT/DELETE /finance/budgets/{id}`, `GET .../expense-classes-totals` | `finance.budgets.*`, `finance.budgets-expense-classes-totals.collection.get` |
| `POST /finance/budgets/{id}/recalculate` | `finance.budgets-recalculate.item.post` |
| `GET/POST /finance/groups`, `/groups/{id}`, `GET /groups/{id}/expense-classes-totals` | `finance.groups.*`, `finance.groups-expense-classes-totals.collection.get` |
| `GET /finance/group-fiscal-year-summaries` | `finance.group-fiscal-year-summaries.collection.get` |
| `GET/POST /finance/group-fund-fiscal-years`, `POST .../batch` | `finance.group-fund-fiscal-years.*`, `.batch.post` (batch endpoint: MODFIN-442 / UIF-629) |
| `POST /finance/allocations` | `finance.allocations.item.post` |
| `POST /finance/transfers` | `finance.transfers.item.post` |
| `POST /finance/release-encumbrance/{id}`, `POST /finance/unrelease-encumbrance/{id}` | `finance.release-encumbrance.item.post`, `finance.unrelease-encumbrance.item.post` |
| `GET /finance/transactions`, `/transactions/{id}`, `POST /finance/transactions/batch-all-or-nothing` | `finance.transactions.collection.get` / `.item.get` / `.batch.execute` |
| `GET/POST /finance/ledger-rollovers`, `GET /ledger-rollovers/{id}`, `-logs`, `-budgets`, `-errors` (GET/POST/DELETE), `-progress` (GET/POST/PUT) | `finance.ledger-rollovers*.…` |
| `GET /finance/fiscal-year-hierarchy` | `finance.fiscal-year-hierarchy.collection.get` (browse tab data, UIF-544..548) |
| `GET /finance/finance-data`, `PUT /finance/finance-data` | `finance.finance-data.collection.get` / `.collection.put` (batch allocation worksheet / Batch edit; PUT supports preview/dry-run — MODFIN-382, Karate `finance-data.feature`) |
| `GET/POST/PUT/DELETE /finance/fund-update-logs[/{id}]` | `finance.fund-update-logs.*` (batch allocation logs, MODFIN-379) |
| `GET /finance/exchange-rate`, `/calculate-exchange`, `POST /calculate-exchange-batch` | `finance.exchange-rate.item.get`, `finance.calculate-exchange.item.get` / `.collection.get` |
| `GET/POST /finance/expense-classes[/{id}]`, `GET /finance/fund-types…`, `GET /finance/fund-codes-expense-classes` | `finance.expense-classes.*`, `finance.fund-types.*`, `finance.fund-codes-expense-classes.collection.get` |

mod-finance-storage extras: `POST /finance-storage/budgets/batch`, `POST /finance-storage/funds/batch` (MODFISTO-517/501), `GET /finance-storage/transaction-totals` and `POST .../transaction-totals/batch` (MODFIN-444), `GET /finance-storage/job-number`, `/finance-storage/exchange-rate-source` (alternate provider, MODFISTO-523), `/finance-storage/settings`, `/finance-storage/budget-expense-classes`, `/finance-storage/group-budgets`, `POST /finance-storage/audit-outbox/process`.

Error codes seen in sources: `currentFiscalYearNotFound` / `plannedFiscalYearNotFound` (UIF-645, MODFIN-445), `failedToCreateAllocations` (cause "Cannot process transactions because a budget is not active or planned", UIF-651), `budgetNotFoundForTransaction`, `notEnoughMoneyForAllocationError`, `budgetRestrictedExpendituresError`, `allowableAllocationIdsMismatch`, `transactionIsPresentBudgetDeleteError`, `budgetExpenseClassReferenceError` (ui-finance en.json keys).

---

## Eureka UI permission → API mapping (ui-finance `package.json`, 9.1.0)

Each UI permission set bundles API permissions; the view set of a record type is a sub-permission of its edit set, edit of create/delete (e.g. `ui-finance.fiscal-year.create` ⊃ `.edit` ⊃ `.view`).

| UI permission (displayName) | Key API permissions |
|---|---|
| `ui-finance.fiscal-year.view/.edit/.create/.delete` | `finance.fiscal-years.*`, `finance-storage.fiscal-year-hierarchy.collection.get` |
| `ui-finance.ledger.view/.edit/.create/.delete` | `finance.ledgers.*`, `finance.finance-data.collection.get`, `finance.group-fund-fiscal-years.batch.post`, `finance.ledger-rollovers-errors.collection.get` |
| `ui-finance.group.view/.edit/.create/.delete` | `finance.groups.*`, `finance.groups-expense-classes-totals.collection.get` |
| `ui-finance.fund-budget.view/.edit/.create/.delete` | `finance.funds.*`, `finance.budgets.*`, `finance.transactions.*`; edit adds `finance.transactions.batch.execute`, inventory location/consortium-search location GETs, `tags.item.post` (fund location restriction in ECS) |
| `ui-finance.allocations.create` | `ui-finance.fund-budget.edit`, `finance.allocations.item.post`, `finance.finance-data.collection.put` (Batch edit / worksheet upload needs this) |
| `ui-finance.transfers.create` | `finance.transfers.item.post` |
| `ui-finance.ledger.rollover.execute` | `finance.ledger-rollovers*`, `tenant-addresses.collection.get` (test-rollover results by email, UIF-653/MODFISTO-509) |
| `ui-finance.fund-budget.recalculate-totals.execute` | `finance.budgets-recalculate.item.post` |
| `ui-finance.encumbrance.release-manually.execute` / `.unrelease.execute` | `finance.release-encumbrance.item.post` / `finance.unrelease-encumbrance.item.post` |
| `ui-finance.fund-update-logs.view` / `.delete` | `finance.fund-update-logs.collection.get/item.get` / `.item.delete` |
| `ui-finance.finance.acquisitions_units.assign.execute` / `.manage.execute` | `finance.acquisitions-units-assignments.create.execute` / `.manage.execute` |
| `ui-finance.settings.view` | read of expense classes, fiscal years, fund types, `finance-storage.settings` |
| `ui-finance.settings.all` | view + expense-class / fund-type CRUD + exchange-rate-source and settings POST/PUT |
| `ui-finance.settings.exchange_rate_source.manage` | `finance-storage.exchange-rate-source.item.get` (UIF-583; UIF-657 renamed permissions in 9.0.0) |
| `ui-finance.settings.exportFundAndExpenseClassCodes` | `finance.fund-codes-expense-classes.collection.get` |
| `ui-finance.exportCSV` | budgets-expense-classes-totals, expense-classes, fiscal-years, fund-types GETs |

TestRail permission checks: C648454 (`User cannot access batch allocation options ... without appropriate capabilities`), C736706 (Exchange Rate Source manage + Finance Settings view cannot configure external exchange rate), C409416 (Settings view sees only appropriate settings). Permission-section cases live under `Permissions --> Finance` (section 180).

---

## Batch Allocation (Allocation tools) — extended (Sunflower → Umbrellaleaf)

> TestRail section 57332 (32 cases). Jira: UIF-525..534, 541, 570, 579, 588, 589, 590, 591, 596, 598, 628, 629, 647, 651, 652, 656, 661; MODFIN-379/381/382/388/403/406/407/412/415/416/419/421/442/443/444; MODFISTO-497/500/501/517/519/554.

- Entry point is **Ledger** `Actions` → `Allocation tools` (`menuSection.allocationTools`) with `Download allocation worksheet (CSV)`, `Upload allocation worksheet (CSV)`, `Batch allocations` (Batch edit budgets) and `Batch allocation logs`. Allocation tools are **not** shown on Fiscal year and Fund panes (C648520, UIF-525) and were removed from the **Group** tab (UIF-598; Group cases in section 57332 are marked DEFERRED, e.g. C648505).
- Batch edit screen title `Batch edit budgets`; buttons `Recalculate` and `Save & close`. Columns (en.json `transaction.allocation.batch.columns.*`): `Fund name`, `Fund status`, `Budget Name`, `Budget status`, `Total allocated (before)`, `Allocation increase/decrease`, `Total allocated (after)`, `Allowable encumbrance %`, `Allowable expenditure %`, `Transaction description`, `Transaction tags`. Cases: C648486, C648530, C651420.
- `Save & close` is disabled if changes were made after clicking `Recalculate` (UIF-591); rows cannot be re-sorted once data is entered (UIF-570); `Unsaved changes` modal on cancel (UIF-652, C1292042); uses an optimized form manager (UIF-628, C1030044 sequential batch allocation with error handling).
- Inline validations (en.json `allocation.batch.form.validation.error.*`): `The following funds are not associated with this ledger (or group) and will NOT be updated.`, `Fund has not been changed and will not be updated.`, `New total allocation cannot be negative.`, `The field must be numeric value.`; recalculation failure `An error occurred while recalculating amounts.`. Success `Allocations have been updated successfully.`; failure `Allocations have not been updated.`; if a budget is not Active/Planned: `Allocations have not been updated because one or more budgets are not active or planned.` (UIF-651, `errors.failedToCreateAllocations`).
- Budget creation rules for funds without budget (MODFISTO-517, UIF-596, C651440, C651433): current FY → new budget status `Active`; future FY → status `Planned` (set automatically once a positive allocation is entered, on both Batch edit and CSV upload); user-assigned valid status is respected. A fund with no budget in the middle of the list must not break the update of other funds (MODFISTO-554, C1045388).
- Fund status side effect (MODFIN-407): increasing/creating allocation > 0 on an `Inactive` or `Frozen` fund makes the fund `Active`; changes that do not increase the allocation, or a decrease that does not go negative, leave fund status unchanged.
- Allocation value `0` is valid (creates a budget with no allocated money; no allocation transaction is generated) — MODFIN-403 comment, UIF-573 comment; negative total allocation is rejected (UIF-576).
- Worksheet upload validation messages (en.json `batchAllocations.uploadWorksheet.*`): modal heading `Upload worksheet` / `Please choose a budget worksheet to upload`; `File has no rows.`; `<file> could not be parsed. Please download the latest worksheet and resubmit.`; `Could not determine the fiscal year from the uploaded file.`; `Inconsistent values found in field: <field>. All rows must have the same value.`; `Invalid file type. Only .csv files are allowed.`; `Missing required header(s): <headers>.`; download failure `An error occurred during file generation.`. Backend rules (MODFIN-403): Budget status text not matching a valid status, Budget UUID / Fund UUID not matching the Fund / Ledger, duplicated rows (fiscalYearId, budgetId, fundId) are rejected; `allocationChange` is optional; nothing is processed if non-editable columns were changed.
- CSV edge cases: commas in fund name/description (UIF-661, 9.0.2), trailing newline (UIF-580), special characters (C648514), unchanged budgets are warned about (UIF-590).
- **Limit: 1000 funds** per ledger for the worksheet and the Batch edit screen (UIF-647, UIF-627; docs to state the limit). TestRail C825374 checks that more than 1000 funds can still be found/displayed in the UI.
- After rollover only **one row per unique budget** appears in the form and worksheet (MODFIN-412, MODFISTO-519, UIF-587; C656288).
- Dry-run/preview: `PUT /finance/finance-data` supports a preview mode (no DB commit, no log written) used by `Recalculate` (MODFIN-382; Karate `finance-data.feature` scenario "Verify PUT finance data with PREVIEW mode").
- **Batch allocation logs** (`Batch allocation logs`, title `Logs`): columns `ID`, `Job name`, `Status`, `Updated records`, `Run by`, `Started running`, `Ended running`; filters `Fiscal Year`, `Ledger`, `Group`, `User`, `Date`; search by `All` / `ID` / `Job name`; delete via `Delete selected logs` → `Delete Log` modal `This action will cause selected logs to be deleted PERMANENTLY. Do you wish to proceed?` → `Log(s) successfully deleted` (single log: `Delete batch allocation log` / `This log will be permanently deleted. Are you sure you want to delete it?`); failure `Log with id {id} was not deleted`. Log ID is unique and not reusable (C651509). Logs of ledgers with acquisition units are hidden from users not in the unit (MODFIN-416, C688780). Cases: C651476, C692072, C651498. "Invalid reference" shown in logs after changing fund status was fixed in 9.0.0 (UIF-656).

---

## Settings (Finance) — extended

- **Fund types**: Settings → Finance → Fund types; create/edit/delete (C6651, C367941); cannot delete a type in use — docs.folio.org Settings_finance: `This fund type cannot be deleted, as it is in use by one or more records.`; optimistic-locking message `You are editing an old version of this data. To load the latest version click cancel, refresh the page and try again.` (docs; TestRail C353951, C356795).
- **Expense classes**: fields `Name`, `Code`, `External account number extension` (en.json `expenseClass.*`); cannot delete when assigned to a budget (docs: `This expense class cannot be deleted, as it is in use by one or more records.`); C15857, C353960.
- **Export fund and expense class codes** (`ui-finance.settings.exportFundAndExpenseClassCodes`): CSV with columns `Fund code` and `Fund and active expense class codes` for a selected FY; only active expense classes (docs Settings_finance; C358967).
- **Exchange rate source** (UIF-581/582/583/584/605/623; MODFISTO-523; MODFIN-410/434): by default exchange rates come from the European Central Bank; checkbox `Specify exchange rate provider` enables fields `Exchange rate provider type` (`No key required` / `Key required`; required), `Exchange rate provider URI` (required), `API Key or Login` (required when key required), `API Secret or Password`; success `Settings successfully updated.`; error `Exchange rate source was not updated.` (en.json `settings.exchangeRateSource.*`). Cases C736705, C736706.
- **Navigation** settings (UIF-544/637/638/639): `Enable browse tab` checkbox enables the Browse tab; success `Settings successfully updated.`; error `Navigation settings were not updated.` (en.json `settings.navigation.*`; draft case C788678 `DRAFT | Enable and disable browse tab in Finance app`).
- Finance Settings view-only user sees only appropriate settings (C409416, UIF-583).

---

## Ledger / Fiscal year / Budget display and calculation rules (2026-11-24)

- Ledger detail has a `Fiscal year` select (current default, previous years selectable) replacing the key/value `Current fiscal year` (UIF-600). Fiscal year pane shows the currency (UIF-538) and Credited totals appear in FY / Ledger / Group / Budget views (UIF-494..497, MODFIN-371/MODFISTO-481 separated credits from expenditures).
- Negative values render in parentheses (UIF-540).
- Fund/budget `Unassigned` expense class row shows totals for transactions without an expense class (UIF-617, MODFIN-433, en.json `budget.expenseClasses.unassigned.tooltip`: `"Unassigned" applies to transactions completed prior to the inclusion of expense classes on the budget, if there are any that exist.`; C15857 area; Cypress `totals-for-transactions-without-expense-class-are-included-in-expense-class-summary.cy.js`). Percent of total expended must stay correct when over-expended (MODFIN-372, C496115, C440081).
- Over-encumbrance is calculated per budget/ledger/FY (MODFIN-373, MODFISTO-484, C496175, C496145/149/153).
- Budget validation on update (MODFIN-374; C503053): remaining-allowed-encumbrance / expenditure checks take paid and pending credits into account (C496166, C496167).
- Fiscal year currency: edit of an FY must not overwrite its currency with the current system currency (MODFIN-386); Cypress `currency-consistency-for-fiscal-year-and-budget-after-changing-currency-settings.cy.js`. Period End Date is end of the selected day, UTC (UIF-666); FY form/view shows `UTC` (Cypress `utc-is-displayed-when-create-or-view-fiscal-year.cy.js`).
- When the ledger's current FY lapses (end date moved to the past) the Ledger/Group view shows `No current fiscal year found.` (`errors.currentFiscalYearNotFound`, MODFIN-445, UIF-539, UIF-645) and no `Rollover is complete` toast appears (C514909); Group details must not throw a null-pointer if the ledger has no active FY (Cypress `no-null-pointer-exception-in-group-details-if-ledger-has-no-active-fiscal-year.cy.js`).
- Budget creation: a negative `Allocated` value shows the inline error `Should be positive` under the field after Save & close (UIF-573 description); negative allowable percentages are also rejected; `0` is allowed (UIF-573 comment, MODFIN-405, C663288). Creating a Planned budget requires an upcoming FY in the same series (C353528).
- Budget deletion messages: with transactions `Budget can not be deleted, because it has transactions`; with expense classes see Error table above. Budgets that have only allocation transactions (from/to) can be deleted (Karate `budget-can-be-deleted-if-have-only-allocation-transactions-from-or-to.feature`; Cypress `delete-budget-with-transfer-transaction.cy.js`).
- Transfers: `Swap` From/To (UIF-524, C808443, UIF-667); transfer in a prior FY shows the correct FY in the toast (UIF-568, C692240); unsuccessful transfer gives `{amount} was not successfully transferred to the budget <b>{budgetName}</b>.` (`transaction.transfer.genericError`, UIF-619, C825296); transfer/allocation with a fund that has no budget: `{amount} was not successfully transferred because <b>{fundCode}</b> has no budget.` (`transaction.transfer.budgetNotFoundForTransaction`; allocation variant `...allocated because...`). Money transfer between funds of different ledgers works when source budget has 0.00 allocation (C535516).
- Cancel on Increase/Decrease allocation modal must not shift focus or trigger form validation (UIF-610); `Move allocation` modal must stay open on an `Amount` error (UIF-608).

---

## Rollover — additional rules and edge cases (2026-11-24)

- Test rollover (`Test rollover`) vs common rollover: Test and common rollover both run for a Planned budget created by the user (C366540); several previews allowed for the same ledger/FY (C359604); test rollover is blocked once the common rollover was done for that ledger and FY (C402327); test rollover results are exported as CSV and can be sent by email (C365633, MODFISTO-509 / UIF-653 `GET/PUT /tenant-addresses`); repeated download of an old test rollover returns that run's data (C366528). Rollover logs: `Rollover logs` action, tabs `Settings`/`Results`/`Errors`, columns include `Start time`, `End time`, `Status`, `Source` (en.json `ledger.rollover.logs.*`); no 10-record limit on results/errors (C380629, C380631); export of results when the fund has no budget works (C770425, UIF-607) — `Failed to export rollover logs` on failure; results CSV shows rounded values when "Adjust allocation, %" is used (C831977, MODFIN-439); error CSV fund code / PO line number columns are known to be empty (UIF-624, C399078 "NOT implemented").
- Duplicate rollover error text (en.json `ledger.rollover.error.conflict`): `<b>{ledgerName}</b> was already rolled over from the fiscal year <b>{fromFYcode}</b> to the fiscal year <b>{toFYcode}</b>`; generic `Error on executing rollover process`. FYs with the same alpha prefix; consecutive rollovers across three FYs (C407710, Karate `ledger-fiscal-year-sequential-rollovers.feature`).
- Encumbrance type toggles (MODFISTO-559/560, C1347127): with no encumbrance option selected, new encumbrances are `Released` with $0.00 and the PO line keeps pointing at the current-FY encumbrance; with only some order types selected (not all, not none), encumbrances for selected types are created `Unreleased` and the PO line link is updated, and encumbrances of the unselected types follow the "none selected" behavior. Test each type combination separately (Ongoing / Ongoing-subscription / One-time).
- Rollover `Based on` = `Remaining` / `Expended` / `Initial encumbrance` with paid invoices: C400643, C375266, C375267, C503142 (credit invoice exists, MODFISTO-483), C436766 (PO line total estimated price rounded), C514851/C514956 (foreign-currency orders; exchange rate set manually).
- Multi-ledger PO lines (two fund distributions in different ledgers, same FY): C397990, C398023, C399059; after rolling both ledgers the budget `Encumbered` of the first-rolled ledger must be correct without `Recalculate budget totals` (MODFISTO-549, fixed 8.8.7).
- Re-encumber OFF: fund distribution of the PO line can be deleted / changed / added after rollover (C451634, C451636, C451639).
- Updating encumbrance after successful rollover: C409478; order-line encumbrance link moves to the current-FY encumbrance (C552995); rollover does not fail on duplicate encumbrance (C446126, MODFISTO-461, Karate `ledger-fiscal-year-rollover-fail-resistance-when-duplicate-encumbrance.feature`); more than 401 related PO lines (C380762); Rollover of budgets increased by percentage must not store more than 2 decimals (MODFIN-439).
- `Rollover transfer` amounts count into budget `Net transfers` after `Recalculate budget totals` (MODFIN-394, C627392).
- Rollover settings screens: `Displaying FY rollover settings` for common (C376605) and test (C376978) rollover; the `Enforce all budget expenditure limits during rollover` toggle was removed from the form and from log settings (UIF-611); the encumbrance-limits checkbox state persists (UIF-537, C356411).
- Unpaid invoices: dialog title `Unpaid invoices` with `Export list` (success `Invoice export has started successfully`, failure `Failed to export invoices`) — C357988, C358143, en.json `invoice.unpaidInvoices.*`.

---

## Transactions and limits — additional cases

- Order cannot be opened when encumbrance exceeds remaining allowed encumbrance (C449364, C503053), can be opened when close to the limit (C449361); invoice cannot be paid when exceeding remaining allowed expenditures (C449368, C449373, C496166, C496167) and can be approved near the limit (C449367). These are Orders/Invoices-side symptoms of the Finance limits.
- Transactions list: filters include `Encumbrance status` (UIF-535); sort and filter (C934300); list is sorted on load (UIF-555); page title for `Rollover transfer` and other detail panes (C451615).
- Unrelease/release encumbrance: `Unrelease encumbrance` is hidden for a closed-order encumbrance (C436937, UIF-503); an unreleased encumbrance rolls over (C436938). BE rule: unrelease is prevented when the encumbrance has pending payment, credit or expended transactions (MODFISTO-542 comment). Cancelling an approved invoice must restore the right encumbrance amount (MODFISTO-537/542, MODFISTO-488/512: credits and cancelled invoices); error toasts: `Error while releasing encumbrance`, `All expected transactions already processed` (release), `Error while unreleasing encumbrance`.
- Recalculate failure toast: `Budget totals could not be updated based on current budget transactions` (en.json `budget.actions.recalculateTotals.error.genericError`); API-level check C432321 (Postman, `Verify work of budget recalculation`).

---

## ECS / consortium notes

- Finance data is tenant-local; **fund location restriction** in ECS lists locations from the affiliations the user belongs to (UIF-561, UIF-606) using `consortium-search.*` and consortia publications permissions in `ui-finance.fund-budget.edit`. Cases (section 36513 `Consortium (Finance)`): C466311 (create/edit restricted fund in Central), C468164 (Member), C468166 (remove location in Central), C468171 (unassign all locations), C877123 (DRAFT: `Summary` column title/description on the Transactions page for a shared Instance, UIF-375/381, MODFIN-251). "Invalid reference" instead of location names in ECS was fixed (UIF-566).
- Browse tab and Navigation setting are not ECS-specific; verify per tenant.

---

## Optimistic locking

Enabled on Budget, Budget+Expense class, Fund, Fiscal year, Fund type, Ledger, Group_Fund_Fiscal_year, Expense classes and Groups tables (TestRail section 17994: C353947–C353961, C356795; MODFISTO-303, UIF-379). In the UI the conflict is shown as a banner (MODFISTO-513, UIF-565 for Groups); Karate `budget-and-fund-optimistic-locking.feature`.

---

## Existing automation coverage (2026-11-24)

**Karate** — `folio-integration-tests` → `acquisitions/src/main/resources/thunderjet/mod-finance/` (runner `acquisitions/src/test/java/org/folio/FinanceApiTest.java`, `@FolioTest(team = "thunderjet", module = "mod-finance")`): `features/` — `finance-data`, `fiscal-year-hierarchy`, `fiscal-year-totals`, `ledger-totals`, `group-fiscal-year-totals`, `group-expense-classes`, `budgets-totals-calculation`, `budget-update`, `budget-expense-classes`, `budget-transfer-transactions`, `budget-and-fund-optimistic-locking`, `allowable-encumbrance-and-expenditure-restrictions`, `create-planned-budget`, `create-inactive-budget`, `current-budget-for-fund`, `curr-fiscal-year-for-ledger-consider-time-zone`, `recalculate-budget`, `unrelease-encumbrance`, `update-encumbrance-transactions`, `batch-transaction-api`, `group-and-ledger-transfers-after-rollover`, budget-deletion cases (`budget-can-not-be-deleted-…`, `budget-can-be-deleted-if-have-only-allocation-transactions-from-or-to`), rollover cases (`ledger-fiscal-year-rollover-MODFISTO-247`, `…-rollovers-multiple`, `…-sequential-rollovers`, `…-preview-rollover`, `…-preview-rollover-need-close-budgets`, `…-skip-previous-year-encumbrance`, `…-pol-and-system-currencies-are-different`, `…-fail-resistance-when-duplicate-encumbrance`, `…-order-with-broken-encumbrance`, `unopen-order-after-rollover-MODORDERS-542`), `acq-units/verify-get-funds-*`; reusable steps in `reusable/` (createFund, createBudget, rollover, createEncumbrance, createPayment…). Not found in Karate: `fund-update-logs` API, exchange-rate-source, worksheet CSV upload endpoints (only `finance-data` PUT/GET).

**Cypress** — `stripes-testing/cypress/e2e/finance/` (~177 specs): `funds/` (create fund, location restriction ×8, allocations/transfers/move/swap, budget create/delete/recalculate, expense classes, donors, tags, acq units), `ledgers/` (create, search, enforce-limits checkboxes, export budget columns incl. credited, summary calc, acq units, `ledger-export-settings-pairwise-variants/`), `groups/` (create, add/remove funds, financial summary after allocation change, pagination in Select funds), `fiscalYears/` (create/view/search, currency consistency, UTC, rollover specs incl. `fiscalYearRollover/` ×29 — logs, test rollover, settings, based-on variants, 2-fund POL, foreign currency), `batchAllocation/` (batch-edit-budgets, download-allocation-worksheet, create-and-update-budget-for-future-fy, recalculate, delete logs, allocation-tools-not-available, only-one-row-after-rollover, update-allocation-for-current-fy-and-create-budget-for-future-fy…), `transactions/` (sort/filter), root-level unrelease-encumbrance and available-balance specs; settings specs in `cypress/e2e/settings/finance*` (expense classes, fund types, export codes, view-only). Fragments: `cypress/support/fragments/finance/**`, `…/settings/finance/**`; UI strings asserted in `cypress/support/fragments/finance/states.js`.

---

## Docs-sourced field semantics (docs.folio.org `Acquisitions/Finance/_index.md`, `Settings_finance.md`)

- **Allowable expenditure %** applies to the budget balance including pending payments but ignoring encumbrances; blank = unrestricted; 100 restricts to available balance; >100 allows beyond it; 0 restricts to 0. **Allowable encumbrance %**: same semantics; only effective if the ledger has `Enforce all budget encumbrance limits` checked.
- **Allocated (required)** on budget create: `0` accepted and generates no allocation transaction.
- Fund `Currency` is prefilled from the FY currency (the FY gets the tenant primary currency at creation — check Settings → Tenant → Language and localization before creating an FY).
- Fund `Restrict use by location` exposes `Locations` accordion; on a PO line only the fund's locations are offered in the location dropdown; to unrestrict also uncheck the box.
- Fund `Transfer from` blank = any fund may transfer in; `Transfer to` blank = may transfer to any fund.
- Fiscal year summary does not show Net transfers (transfers only within one FY); Net transfers appear on Ledger, Group and Budget summaries.

---

## Known Gaps / Items to Verify

- [ ] Jira scope: no fixVersion exists for Umbrellaleaf R2 2026 on UIF/MODFIN/MODFISTO yet (latest seen: UIF 9.0.0, MODFIN 5.2.0, MODFISTO 8.9.0 = Trillium R1 2026; master is UIF 9.1.0 / MODFIN 6.0.0-SNAPSHOT); items after 2026-04 were scoped by resolution date (UIF-544, UIF-666, MODFIN-465, MODFISTO-565).
- [ ] Browse tab (UIF-544..548): commits showed revert and re-introduction in Aug–Oct 2026; settings page and `browse.*` strings exist in en.json; only a DRAFT TestRail case (C788678) — verify in target env before asserting.
- [ ] Rollover confirm dialog text differs between `en.json` (current) and `en_US.json`/old file; this file now uses en.json. The `FOLIO has found invoices that are not yet paid...` unpaid-invoice message is unchanged.
- [ ] Docs (docs.folio.org) list Frozen/Inactive budget statuses and say a budget can be "Frozen"; older TestRail cases mention only Active/Planned/Closed — expected behavior of Frozen/Inactive budgets for transactions is not documented in any source read.
- [ ] Rules 'Deleting Records' table (fund/ledger/group/FY delete conditions) and 'Over encumbered/over expended only shows if ledger restrictions disabled' have no source in this refresh; keep conservative.
- [ ] `en.json` exposes `transaction.transfer.budgetRestrictedExpendituresError` with different wording (`{amount} could not be transferred to the budget <b>{toBudgetName}</b> because it exceeds the allowable expenditure limit for <b>{fromBudgetName}</b> and ledger fund restrictions are active.`) than the `states.js`-sourced string in the Error table; check which one the UI renders per case.
- [ ] Confluence (folio-org.atlassian.net/wiki anonymous REST) returned only Acquisitions SIG meeting notes / RM SIG pages for "fiscal year rollover"; no Finance requirement page was found or relied on.
- [ ] Jira Done search returned no entries with fixVersion naming Umbrellaleaf/Trillium/Sunflower/Ramsons by release name (fixVersions are module versions); mapping used: Ramsons R2 2024 = UIF 7.0.0 / MODFIN 5.0.0 / MODFISTO 8.7.0; Sunflower R1 2025 = UIF 8.0.0 / MODFIN 5.1.0 / MODFISTO 8.8.0; Trillium R1 2026 = UIF 9.0.0 / MODFIN 5.2.0 / MODFISTO 8.9.0 (from "Release" tasks UIF-558/599/660, MODFIN-383/420/459, MODFISTO-505/521/556).
- [ ] Batch allocation worksheet column list beyond the ones confirmed in C648502 was not re-verified against the BE schema (`finance-data.json` not read).
- [x] "Exact UI Texts" seeded 2026-07-21 and refreshed against en.json 2026-11-24 (batch allocation, upload validation, logs, exchange rate, navigation, rollover options added).
- [x] The old note that Jira returned 0 Done issues for component "Finance" is obsolete: project keys UIF / MODFIN / MODFISTO are used (93 / 63 / 69 Done issues resolved since 2024-06-01).

> Random spot-check (2026-07-22): picked one fresh uncited case at random from the "Fiscal Year Rollover" subsection (C376609) — a large, detailed rollover scenario. The general "Planned budget is created" rule already existed (21) but didn't cover the specific zeroed-out outcome when Rollover allocation is off and Rollover budget value is None. Added Rule 30 as a refinement rather than a new top-level fact.


---

## Changes since 2026-07-21 (delta refresh 2026-11-07)

> Delta, not a rebuild: sections above reflect the original build. Where this section contradicts them, **this section wins**.
> Sources: GitHub folio-org/ui-finance (translations/ui-finance/en.json, HEAD v9.0.0); Jira folio-org.atlassian.net Done issues resolved since 2026-07-21; TestRail suite 21 cases created since 2026-07-21.

### What changed
- **Browse** tab in Finance (Search | Browse): select a fiscal year, hierarchy Ledger → Group → Fund → Budget → Expense class, status filters (Ledger/Group/Fund/Budget/Expense class status), New fiscal year/ledger/group/fund actions (UIF-545..548; enabled via finance navigation setting UIF-544). Commits show a revert on 2026-08-18 and re-introduction in Sep–Oct — verify the tab is present in the target env before asserting.
- Fiscal year **Period End Date** displays as the end of the selected day, labelled UTC (UIF-666).

### UI strings changed since 2026-07-21 [source — en.json, 2026-11-07]

> `en.json` is the source of truth; `en_US.json` is a Lokalise copy that can lag behind. `<b>`/`<strong>` render as bold; `{x}` are placeholders.

**New strings**

| Key | Text |
|---|---|
| `browse.tab.search` | Search |
| `browse.tab.browse` | Browse |
| `browse.title` | Browse finance |
| `browse.subtitle` | Select a fiscal year to browse results |
| `browse.selectFiscalYear` | Select a fiscal year to browse results. |
| `browse.filter.fiscalYear` | Fiscal years |
| `browse.filter.ledgerStatus` | Ledger status |
| `browse.filter.groupStatus` | Group status |
| `browse.filter.fundStatus` | Fund status |
| `browse.filter.budgetStatus` | Budget status |
| `browse.filter.expenseClassStatus` | Expense class status |
| `browse.actions.newFiscalYear` | New fiscal year |
| `browse.actions.newLedger` | New ledger |
| `browse.actions.newGroup` | New group |
| `browse.actions.newFund` | New fund |

### Jira — Done since build (finance (UIF, MODFIN) since 2026-07-21)

- UIF-666 Bug — Fiscal year Period End Date displayed as end of selected day (23:59 UTC)
- (GitHub only, no Done ticket yet) UIF-544..548 Browse tab / finance hierarchy

### TestRail — cases created since 2026-07-21: 1 (1 manual, 0 automated/Karate)

- C1504439 Create and update budget via upload allocation worksheet (CSV) for the Current fiscal year from Ledger view — refs UIF-526,UIF-527,UIF-528,UIF-529,UIF-591,MODFIN-419,UIF-628,UISACQCOMP-283,MODFIN-443,MODFIN-442,MODFIN-444
