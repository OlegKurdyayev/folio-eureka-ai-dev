# FOLIO Finance App - Business Logic Context

> Built: 2026-07-21 · Delta-refreshed: 2026-11-07 · Refreshed: 2026-11-25 · Release: Umbrellaleaf R2 2026 (ui-finance master package.json 9.1.0; mod-finance master 6.0.0-SNAPSHOT; last fixVersions seen: UIF 9.0.0 / MODFIN 5.2.0 / MODFISTO 8.9.0 = Trillium R1 2026) · Builder: build-app-context/manual — header normalized 2026-11-07, see area-map.md → Freshness

> Originally built 2026-06-18 by build-app-context (TestRail group 454: 12 sections / 249 cases; ui-finance + mod-finance-storage; Jira component "Finance" returned 0 Done issues, so UIF / MODFIN / MODFISTO keys are used since the 2026-11 refresh).
> **Manual enrichment 2026-07-21:** TestRail C186156 / C380517 / C648502 (transaction detail pane, budget structure, rollover page/dialogs) and `folio-org/ui-finance` `translations/ui-finance/en_US.json` (toasts, errors, confirm dialogs, form labels). `en.json` is now the source of truth for strings (see Known Gaps).
> Combines stable Finance domain rules with run evidence for test-writing decisions.

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
- The "current" fiscal year is determined by dates, not by a manual flag: mod-finance picks it from the ledger's fiscal-year series using today's date in the system time zone (`LedgerDetailsService.defineCurrentFiscalYear`; Karate `curr-fiscal-year-for-ledger-consider-time-zone.feature`)
- A ledger can be associated with multiple fiscal years (for rollover history)
- Budget currency is locked at fiscal year creation time - set to the tenant primary currency when the fiscal year record is created; if tenant currency changes later, existing budgets remain in the original currency (docs.folio.org Finance → Fiscal year `Currency`; MODFIN-386, UIF-538)
- Optional: Acquisition units, Description
- Before creating an FY check Settings → Tenant → Language and localization `Primary currency` - the FY (and its budgets/funds) take that currency (docs Finance → Fiscal year)

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
| Enforce all budget encumbrance limits | Checked by default (docs Finance → Ledger; `CreateLedger.js` initial `restrictEncumbrance: true`); if checked: FOLIO rejects encumbrances that exceed available budget |
| Enforce all budget expenditure limits | Checked by default (docs; `CreateLedger.js` `restrictExpenditures: true`); if checked: FOLIO rejects expenditures that exceed available budget |
| Acquisition units | Controls who can view/edit this ledger |
| Description | Optional |

### Ledger status values (docs.folio.org Finance → Ledger; schema `ledger.json`)
- Active - ongoing
- Frozen - paused (docs say only "put on pause"; transaction blocking for Frozen is (unverified), see Known Gaps)
- Inactive - no longer in use

### Ledger detail view
Shows: associated Funds with current Budget values, Groups, summary totals per fiscal year.

### Exporting ledger funds and budgets
- Ledger detail view -> Actions -> Export -> downloads CSV of all funds and budgets for the ledger

### Rollover
Triggered from ledger Actions -> Rollover:
See "Rollover Fiscal Year" and "Rollover - additional rules" below for what it does. It does not advance the "current" FY - that is date-driven.
- Requires `Procedural - UI-Finance Ledger Rollover - Execute`

---

## Group

### Required fields
- Name
- Code
- Status - Active / Frozen / Inactive

### Key characteristics
- Cross-ledger grouping of funds for summary/reporting
- Does not restrict transactions - summary/reporting only (docs Finance → Group: groups bring funds together so totals can be summarized by group, ledger or fiscal year)
- A fund can belong to multiple groups (docs Finance → Group)
- Optional: Acquisition units, Description

---

## Fund

### Required fields
- Name
- Code - must be unique; alphanumeric recommended
- Ledger - exactly one (docs.folio.org Finance → Fund `Ledger`); "cannot be changed after creation" is (unverified) - the Fund form (`ui-finance` `FundForm.js`) does not disable the Ledger select in edit mode, see Known Gaps
- Status - Active / Frozen / Inactive

### Key fields
| Field | Notes |
|---|---|
| Currency | Pre-filled from the fiscal year currency; read-only after creation |
| Type | Optional category (e.g. Endowment, Restricted); used to group funds at rollover (docs Finance → Fund `Type`); funds without type show `No fund type` on the Rollover page (`ledger.rollover.noFundType`, `RolloverLedgerBudgets.js`) |
| Acquisition units | Controls who can edit the fund (docs Finance → Fund); hiding restricted funds from fund-distribution dropdowns in Orders/Invoices is (unverified) - Orders/Invoices-side, not checked |
| Group | Assign fund to one or more groups |
| Transfer from | Restrict which funds can transfer into this fund; blank = any fund allowed (docs Finance → Fund) |
| Transfer to | Restrict which funds this fund can transfer to; blank = any fund allowed |
| External account number | Optional accounting system reference |
| Description | Optional |
| Donor information | Optional donor attribution |
| Locations | Optional location restrictions - limit fund use to specific library locations (docs: appears when `Restrict use by location` is checked; on a PO line only the fund's locations are offered in the location dropdown; to unrestrict, also uncheck the box) |
| Tags | Optional |

### Fund status
- Active - required for orders to be opened against this fund and for invoices to be fully paid (docs Finance → Fund `Status`)
- Frozen - paused (docs); Inactive - no longer in use (docs). Transaction blocking for Frozen/Inactive is (unverified), see Known Gaps

---

## Budget

One budget per Fund per Fiscal Year.

### Required fields
- Name (auto-suggested from fund code + fiscal year code)
- Fiscal Year - selects which FY this budget covers

### Key fields
| Field | Notes |
|---|---|
| Allowable encumbrance (%) | Maximum percentage of allocation that can be encumbered; optional; if blank, no encumbrance limit beyond ledger setting (docs Finance → Budget; semantics under Docs-sourced field semantics) |
| Allowable expenditure (%) | Maximum percentage of allocation that can be expended; optional |
| Status | Active / Planned / Closed / Frozen / Inactive (schema `budgetStatus` — acq-models `budget.json`; docs Finance → Budget status; C663288 creates Inactive and Closed budgets) |
| Expense classes | Add one or more expense classes to track spending by category within the budget |

### Budget financial buckets (docs.folio.org Finance → Budget summary; formulas also verified in cases C449364, C496145)

| Bucket | Description | Formula |
|---|---|---|
| Allocated | Total money assigned to this budget | Sum of all Allocation transactions |
| Net transfers | Money moved in/out via Transfer transactions | Transfers in - Transfers out |
| Total funding | Allocated + Net transfers | Allocated + Net transfers |
| Encumbered | Reserved for open orders not yet invoiced | Sum of open Encumbrance transactions |
| Awaiting payment | Approved invoices not yet paid | Sum of Pending payment transactions |
| Expended | Paid invoices | Sum of Payment transactions |
| Credited | Sum of Credit transactions | Shown separately from Expended (docs Finance → Budget summary; UIF-494..498) |
| Unavailable | Encumbered + Awaiting payment + Credited + Expended | Sum of all committed/spent amounts (docs: credited is part of Unavailable) |
| Over encumbered | Amount by which encumbrances exceed available (docs Finance → Fiscal year summary) | Only shows if ledger restrictions disabled (unverified) |
| Over expended | Amount by which expenditures exceed available | Only shows if ledger restrictions disabled (unverified) |
| Cash balance | Total funding - Expended | Remaining cash |
| Available | Total funding - Unavailable | Money not yet committed |

### Budget status values
Schema enum (acq-models `budget.json`): `Active`, `Frozen`, `Inactive`, `Planned`, `Closed`.
- Active - current; accepts transactions (docs Finance → Budget `Status`)
- Planned - future budget; not yet active for transactions (FOLIO still allows a Planned budget in the current FY and an Active one in a future FY — MODFIN-407 comment)
- Closed - the budget's period of activity has ended (docs: typically a result of fiscal year rollover); docs state the budget must be Active to open an order or approve/pay an invoice
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
- Requires `Data - UI-Finance Allocations - Create` + the Fund-Budget Edit capability set (docs: must include view and edit fund and budget)

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
- Batch allocation jobs are logged; logs can be viewed and deleted (separate capability sets: `Data - UI-Finance Fund-Update-Logs - View` / `- Delete`, docs)

#### Allocation worksheet (CSV) — download & upload (confirmed C648502/C648514/C651511 etc.; refs UIF-525, UIF-526, MODFIN-391, MODFIN-421)

- **Download:** from a **Ledger view** → `Actions > Download allocation worksheet (CSV)` → **"Select fiscal year"** modal (`Fiscal year` dropdown, **newest FY selected by default**, `Cancel` / `Confirm`). Confirm shows toast `Please wait while the worksheet is generated. Your download will start automatically` and downloads a CSV.
- **Worksheet columns:** Fund columns are always populated — `Fiscal year`, `Fund name`, `Fund code`, `Fund UUID`, `Fund status`; Budget columns — `Budget name`, `Budget UUID`, `Budget status`, `Budget initial allocation`, `Budget current allocation`, … — are **blank when the fund has no budget for that FY** (that's how a budget gets *created* on upload).
- **Upload:** `Actions > Upload allocation worksheet (CSV)` **creates or updates budgets in batch** for the worksheet's fiscal year — works for both the **Current** and a **Future** fiscal year (C648514 / C651511). A blank budget row → budget created; a filled row → budget updated.
- **Upload validations (each its own negative case; exact messages and rules in the Batch Allocation section):** non-`.csv` file, missing column, CSV not matching the generated worksheet, differing fiscal years, fund of another Ledger, invalid fund/budget status. Access is capability-gated (`ui-finance.allocations.create`; C648454).

---

## Transfer Transactions

Transfers move money between two fund budgets.

### Creating a transfer
- From budget: Actions -> Transfer
- Specify: From fund (source), To fund (target), Amount, Fiscal year
- Transfer restrictions on funds (Transfer from/to fields) are enforced
- Requires `Data - UI-Finance Transfers - Create` (+ Fund-Budget Edit per docs)

### Transfer restrictions
- If a fund has Transfer from restrictions: only listed funds can transfer into it
- If a fund has Transfer to restrictions: only listed funds this fund can transfer to
- Blank fields = no restrictions

---

## Encumbrances

Created automatically by Orders when a PO is opened (docs Finance → Transactions filter descriptions); no create action exists in the Finance UI (mod-finance exposes no dedicated encumbrance POST - only `POST /finance/transactions/batch-all-or-nothing`, see API Surface).

### Manually releasing an encumbrance
- Budget transaction log -> select encumbrance -> Release encumbrance
- Requires `Procedural - UI-Finance Encumbrance Release-Manually - Execute` (see Capability Sets)
- Use case: vendor cancelled; free funds before invoice arrives

### Unreleasing an encumbrance
- Budget transaction log -> select released encumbrance -> Unrelease encumbrance
- Requires `Procedural - UI-Finance Encumbrance Unrelease - Execute` (see Capability Sets)
- Use case: encumbrance released by mistake during invoice approval

---

## Transactions

All financial activity is recorded as transactions. mod-finance-storage exposes only `GET` and `POST /finance-storage/transactions/batch-all-or-nothing` for transactions (no PUT/DELETE; descriptor `finance-storage.transactions`), so there is no user-facing edit/delete; internal changes (e.g. encumbrance release/update) go through the batch endpoint.

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
- User must be a member of at least one assigned unit to edit the record; if no units are assigned any user with the capability can edit it (docs Finance → Fiscal year/Ledger/Group/Fund `Acquisition units`)
- Users with a bypass-acquisition-units capability can access all records (unverified - no Finance source read)
- Hidden from fund distribution: funds with acquisition unit restrictions are filtered out of fund selection dropdowns in Orders/Invoices for users outside those units

### Assign on create vs manage on edit
- Assign acquisition units to new record (Procedural execute) - only allows assigning units on create
- Manage acquisition units - allows changing unit assignment when editing existing records (docs table lists it as Data manage; package.json names it `ui-finance.finance.acquisitions_units.manage.execute`: type unresolved, see Capability Sets). `FundForm.js` uses `CREATE_UNITS_PERM` on create and `MANAGE_UNITS_PERM` on edit.

---

## Recalculate Budget Totals

In case of data inconsistency (e.g. transactions deleted administratively):
- Budget -> Actions -> Recalculate budget totals
- Recomputes all bucket values from actual transactions (`POST /finance/budgets/{id}/recalculate`; docs capability description: use after financial transactions were deleted)
- Requires `Procedural - UI-Finance Fund-Budget Recalculate-Totals - Execute`

---

## Deleting Records

Verified general rule (docs.folio.org Finance → notes on deleting fund/ledger/group/budget records): financial structure records cannot be deleted if other records are assigned to them; budgets cannot be deleted once they have any transaction other than allocations (Karate `budget-can-be-deleted-if-have-only-allocation-transactions-from-or-to.feature`); funds cannot be deleted if they were part of a fiscal year rollover, including test rollovers. Error texts: see Error table.

| Record | Per-record condition |
|---|---|
| Budget | No transactions other than allocations (docs); "status must be Planned or Closed" is (unverified) |
| Fund | Not part of a rollover (docs); "no active budget with transactions; not referenced by open orders or unpaid invoices" is (unverified) |
| Ledger | Has no assigned funds (docs, general rule); "no active funds" wording (unverified) |
| Group | "Always deletable" is (unverified) - docs only give the general rule |
| Fiscal Year | Not referenced by a ledger (docs, general rule) |

---

## Rollover Fiscal Year

Triggered from a Ledger -> Actions -> Rollover.

### Rollover configuration options
Exact fields and dropdown values: see `Rollover page — fields` in Exact UI Texts (en.json `ledger.rollover.*`). The earlier generic list ("Allocation strategy: full / remaining / none") was superseded by `Rollover budget value` (`None`/`Available`/`Cash balance`), `Rollover value as` and `Based on`; rollover settings are defined per fund type (docs Finance → Rollover fiscal year).

### What rollover does (docs Finance → Rollover fiscal year)
1. Creates new (Planned) Budget records for each fund in the ledger for the new fiscal year, with or without allocation depending on fund type settings
2. Rolls encumbrances onto the new budgets by order type (one-time / ongoing / ongoing-subscription)
3. Optionally closes current year budgets (status = Closed)
4. Writes a rollover log (results / errors)

### Rollover log
- Viewable after rollover: shows successes, errors, funds processed
- Accessible from: Ledger -> Rollover logs

---

## Exact UI Texts

> Use these verbatim in expected results; never paraphrase. Seeded 2026-07-21 from TestRail C186156 / C380517 / C648502 (modal, transaction pane, budget, rollover fields) and `folio-org/ui-finance` `translations/ui-finance/en_US.json` (toasts, errors, dialogs, labels); refreshed against `en.json` 2026-11-24. `{placeholders}` interpolate at runtime.

### Transaction detail pane — fields (the core Finance verification surface)

When a step opens a transaction (Encumbrance, Payment, Pending payment, Credit, Allocation, Transfer) from a budget's `View transactions`, the detail pane shows these fields — verify column-by-column with exact values [confirmed — C186156]:
`Transaction date`, `Fiscal year`, `Amount`, `Source` (e.g. the PO line, hyperlinked), `Type`, `From` (source fund), `To` (target fund), `Expense class`, `Tags`, `Initial encumbrance`, `Awaiting payment`, `Expended`, `Status`, `Description`.
Encumbrance `Status` values: `Unreleased`, `Released`, `Pending`.

### Budget page structure

- Fund pane accordions: `Current budget`, `Planned budget` (a rolled-over budget appears here named `<Fund code> - <FY code>`, e.g. "Fund A - FY #2"). [confirmed — C186156]
- Budget page accordions: `Budget summary` (Funding information table: Allocated, Net transfers, Total funding, Encumbered, Awaiting payment, Expended, Unavailable, Over encumbered/expended, Cash balance, Available), `Budget information` (contains `Status` field and the `View transactions` link), `Expense classes`.
- Budget `Status` values seen in rollover cases (C186156): `Active`, `Planned`, `Closed`; full list in Status Values table.

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
`Rollover encumbrances` accordion: `Ongoing encumbrances` toggle, `Ongoing-subscription encumbrances` toggle, `One-time encumbrances` toggle, each with a `Based on` dropdown (options `Expended`, `Initial encumbrance`, `Remaining` — en.json `ledger.rollover.basedOn.*`; C375267/C400643 cover `Remaining`; C376601–C376611 cover the option combinations) and an `Increase by, %` field.

> Interaction model for insufficient funds on **Move allocation**: validation is inline and disables `Confirm` (`Total allocation cannot be less than zero`, C380517); there is NO "click Confirm -> error toast" flow for this case. Transfer behaves differently (rule 25).

### Button / Action Labels (key actions)

Actions, Increase allocation, Decrease allocation, Move allocation, Transfer, Confirm, Cancel, Continue, View transactions, Rollover, `Close & view ledger details`, Export (Ledger), Save & close. [from cases]

### Toast Messages

> All rows: **[confirmed — `ui-finance` translations, 2026-07-21]** unless a row names `finance/states.js`; `<b>` marks bold segments.

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

### Error / Validation Messages (confirmed — translations)

| Condition | Message text |
|---|---|
| Allocation/transfer would make total allocation negative | `Total allocation cannot be less than zero` (red text under `Amount`; disables Confirm; C380517) |
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

> Names are `<Type> - <Resource> - <Action>`. Sources: **[Cypress]** = `stripes-testing/cypress/support/dictionary/capabilitySets.js` and `cypress/e2e/eureka/**` role specs; **[docs]** = docs.folio.org Finance → "Capabilities & capability sets" table (resource / type / action); **[pkg]** = ui-finance `package.json` permission ids. Eureka has separate actions View / Edit / Create / Delete (docs: "View, edit" = Edit, "View, edit, create" = Create, "View, edit, delete" = Delete) - there is NO combined `Create/Edit` action.

| Action | Capability Set | Source |
|---|---|---|
| View fiscal years | Data - UI-Finance Fiscal-Year - View | Cypress, docs |
| Edit fiscal years | Data - UI-Finance Fiscal-Year - Edit | docs |
| Create fiscal years | Data - UI-Finance Fiscal-Year - Create | Cypress (`create-role-selecting-apps.cy.js`), docs |
| Delete fiscal years | Data - UI-Finance Fiscal-Year - Delete | docs |
| View / Edit / Create / Delete ledgers | Data - UI-Finance Ledger - View / Edit / Create / Delete | docs (not in Cypress dictionary) |
| View / Edit / Create / Delete groups | Data - UI-Finance Group - View / Edit / Create / Delete | docs (not in Cypress dictionary) |
| View funds and budgets | Data - UI-Finance Fund-Budget - View | Cypress (`uiFinanceFundBudgetView`), docs |
| Edit / Create / Delete funds and budgets | Data - UI-Finance Fund-Budget - Edit / Create / Delete | docs (not in Cypress dictionary) |
| Create allocations | Data - UI-Finance Allocations - Create | Cypress, docs |
| Create transfers | Data - UI-Finance Transfers - Create | Cypress, docs |
| Execute fiscal year rollover | Procedural - UI-Finance Ledger Rollover - Execute | docs |
| Manually release encumbrance | Procedural - UI-Finance Encumbrance Release-Manually - Execute | docs |
| Unrelease encumbrance | Procedural - UI-Finance Encumbrance Unrelease - Execute | docs |
| Recalculate budget totals | Procedural - UI-Finance Fund-Budget Recalculate-Totals - Execute | docs |
| Export finance records | Procedural - UI-Finance - Execute (UI permission `ui-finance.exportCSV`) | Cypress, docs |
| Assign acquisition units to new record | Procedural - UI-Finance Finance Acquisitions Units Assign - Execute (UI permission `ui-finance.finance.acquisitions_units.assign.execute`) | Cypress role specs; docs table spells the resource `UI-Finance Acq Unit Assignment` (docs lag - verify in the target env) |
| Manage acquisition units on existing record | Data - UI-Finance Acq Unit Assignment - Manage (capability set name not confirmed — verify); package.json id `ui-finance.finance.acquisitions_units.manage.execute` suggests a Procedural ... Execute set instead | docs vs pkg conflict |
| View batch allocation logs | Data - UI-Finance Fund-Update-Logs - View | docs |
| Delete batch allocation logs | Data - UI-Finance Fund-Update-Logs - Delete | docs |
| View Finance settings | Settings - UI-Finance Settings - View | Cypress (`create-role-selecting-apps.cy.js`), docs |
| Manage (view, edit, create, delete) Finance settings | Settings - UI-Finance Settings - Manage | docs |
| Export fund and expense class codes | Settings - UI-Finance Settings ExportFundAndExpenseClassCodes - View | docs (Settings_finance) |
| Manage external exchange rate source | Settings - UI-Finance Settings Exchange Rate Source - Manage | docs (Settings_finance; UIF-583) |

Docs' own table has typos (e.g. a missing closing `**` on the Fund-Budget delete row); where docs and Cypress disagree, Cypress (what the automated role specs select in the UI) wins.

---

## Key Business Rules for Test Cases

> Each rule carries its source: docs = docs.folio.org Finance (`Acquisitions/Finance/_index.md`, `Settings_finance.md`); cases = TestRail C-ids; en.json = ui-finance translations; Karate/Cypress = automation paths. Rules about batch allocation, rollover edge cases, settings, display/calculation, optimistic locking and limits are in the dedicated sections below (each bullet there has its Jira/C-id source) and are not repeated here.

1. Budget values (Allocated, Net transfers, Encumbered, Expended ...) are sums of transactions; money enters a budget only through Increase/Decrease/Move allocation, Transfer, Batch allocation or Rollover, not by editing a field. (docs Finance → Budget summary / Allocating money to a budget)
2. Budget currency is locked at fiscal year creation (tenant primary currency at that moment); tenant currency changes do not alter existing budgets, and editing an FY must not overwrite its currency. (docs Fiscal year `Currency`; MODFIN-386, UIF-538; Cypress `currency-consistency-for-fiscal-year-and-budget-after-changing-currency-settings.cy.js`)
3. A fund belongs to exactly one ledger. (docs Fund `Ledger`) Reassigning the ledger after creation is (unverified), see Known Gaps.
4. Ledger `Enforce all budget encumbrance/expenditure limits` are checked by default; when checked, over-limit encumbrances/expenditures are rejected. (docs Ledger; `CreateLedger.js`)
5. Acquisition units gate editing of Fiscal year/Ledger/Group/Fund records (blank = any user with the capability). (docs `Acquisition units` fields) Hiding restricted funds from Orders/Invoices distribution dropdowns is (unverified).
6. A fund must be Active for an order to open or an invoice to be fully paid; the budget must also be Active. (docs Fund `Status`, Budget `Status`)
7. Groups are summary/reporting only and do not affect transaction processing. (docs Group)
8. The current fiscal year is date-driven (series + today's date in the system time zone); rollover does not set it manually. (`LedgerDetailsService.defineCurrentFiscalYear`; Karate `curr-fiscal-year-for-ledger-consider-time-zone.feature`; docs Rollover)
9. No PUT/DELETE exists for transactions in mod-finance-storage (only GET and `batch-all-or-nothing` POST); corrections are done via Recalculate or compensating transactions. (descriptor `finance-storage.transactions`; docs Recalculate)
10. One budget per fund per fiscal year. (mod-finance-storage `schema.json` unique index `budget(fundId, fiscalYearId)`; docs Budget)
11. A budget must be Active for orders to open and invoices to be approved/paid; behaviour of Closed/Frozen/Inactive budgets for other transactions is (unverified), see Known Gaps. (docs Budget `Status`)
12. Fund type drives rollover settings per type; funds without a type appear as `No fund type` on the Rollover page. (docs Fund `Type`, Rollover; `ledger.rollover.noFundType`)
13. Budget `Allowable encumbrance/expenditure %` add limits per budget (blank = unrestricted; encumbrance % effective only if the ledger enforces encumbrance limits). (docs Budget; see Docs-sourced field semantics; C449364)
14. Transfer restrictions (`Transfer from` / `Transfer to`) are enforced in both directions; violation gives `..."allow transfer to or from" Fund setting ... is too restrictive.` (docs Fund; en.json `transaction.transfer.allowableAllocationIdsMismatch`)
15. Assign and Manage acquisition-unit capabilities are distinct; the Fund form uses `CREATE_UNITS_PERM` on create and `MANAGE_UNITS_PERM` on edit. (docs capabilities table; ui-finance `FundForm.js`; package.json `ui-finance.finance.acquisitions_units.assign|manage.execute`)
16. Rollover needs its own procedural capability set `Procedural - UI-Finance Ledger Rollover - Execute`. (docs; package.json `ui-finance.ledger.rollover.execute`)
17. Release and Unrelease encumbrance are separate procedural capability sets. (docs; package.json `ui-finance.encumbrance.release-manually.execute` / `.unrelease.execute`)
18. Batch allocation log deletion (`Fund-Update-Logs - Delete`) is separate from viewing logs (`- View`). (docs; package.json `ui-finance.fund-update-logs.view` / `.delete`)
19. Fund code is unique across the whole tenant (not per ledger); fund name is unique per ledger. (mod-finance-storage `schema.json` unique index `fund(code)`; docs Fund `Code`; en.json `This Fund code is already in use.` / `This Fund name is already in use on the selected Ledger.`; `FundForm.js` name check by ledgerId)
20. Expense classes are optional; transactions without one appear in the `Unassigned` row. (docs Finance → Expense classes; UIF-617, MODFIN-433)
21. **Rollover creates a `Planned budget` for each fund in the new FY** (named `<Fund code> - <FY code>`) and, with `Close all current budgets`, moves the source budgets to `Closed`; encumbrances roll per the Rollover encumbrances toggles and land as `Unreleased` in the new-FY budget (the closed-FY copy keeps the pre-rollover end state). (cases — C186156)
22. **Unpaid invoices block-with-warning**: the `Unpaid invoices` dialog appears and requires an explicit `Continue` before the rollover confirm. (cases — C186156)
23. **A fiscal year series is identified by its alpha code prefix** (`FYA2025` -> `FYA`); rollover requires the next FY to share the same alpha. (cases — C186156; docs Fiscal year `Code`)
24. **A ledger can be rolled over only once from a given source FY.** A second rollover from the now-current FY-B to FY-C fails with `{ledgerName} was already rolled over from the {fromYearCode} fiscal year to the {toYearCode} fiscal year` and does not run; test it as its own scenario (rollover -> succeeds; immediately again -> blocked with this message). Test rollover is likewise blocked after the common rollover. (cases — C360956, C402327; en.json `ledger.rollover.error.conflict`)
25. **Move allocation vs Transfer on negative available**: Move allocation is a hard block (`Confirm` disabled, inline `Total allocation cannot be less than zero`); Transfer allows a negative available balance but first opens a `Negative available amount` dialog (`Completing this transfer will result in <budgetName> having a negative available amount. Are you sure you would like to complete this transaction?`) and only a second `Confirm` commits it. Never write the two with the same behaviour. (cases — C374166, C374183)
26. **Remaining allowed encumbrance** = `(allocated + netTransfers) × allowableEncumbered% − (encumbered + awaitingPayment + expended)`. When a PO's requested encumbrance exceeds it (ledger `Enforce all budget encumbrance limits` active), Open fails with `One or more fund distributions on this order can not be encumbered, because there is not enough money in [{Fund name}].`; the order stays `Pending`, the POL's Current encumbrance shows `-`, and no encumbrance transaction is created (verify via Fund > Actions > View transactions for the current budget). When writing the Orders test, cross-check against this formula, not a round-number threshold. (cases — C449364)
27. **Negative `Available` (and other negative figures) render in parentheses**, e.g. `($500.00)`, since Ramsons; pre-Ramsons screenshots/cases show a leading `-` - do not assert the old format on a current release. (cases — C374183, C496145; UIF-540)
28. **Fund `Restrict use by location` blocks Save & close by itself**: checking it expands `Locations` with `Locations must be assigned` while empty; Save & close with zero locations stays on the page (no toast, no navigation). Unchecking removes the requirement (accordion disappears, save proceeds). Test the blocked path and the unchecked path separately. (cases — C423530)
29. **Rollover based on `Expended` promotes the Planned budget to Active and folds the old Current allocation into it**: old FY budget -> `Closed`; new FY budget -> `Active` with `Initial allocation` = its own pre-set allocation, `Increase in allocation` = old Current allocated amount, `Total allocated` = sum. A paid-invoice amount above the old POL fund-distribution value rolls forward as a new `Unreleased` encumbrance (same PO number, `Initial encumbrance` = that amount, `Awaiting payment`/`Expended` reset to $0.00). Both effects belong to the same scenario. (cases — C359186)
30. **With `Rollover allocation` OFF and `Rollover budget value` = `None` the Planned budget is still created and set `Active`, fully zeroed** (`Initial allocation`, `Increase/Decrease in allocation`, `Total allocated`, `Net transfers`, `Unavailable`, `Available` = `$0.00`/`0`); refines rule 21. (cases — C376609)
31. **Budget `status` has five schema values** (`Active`, `Frozen`, `Inactive`, `Planned`, `Closed`); budgets can be created Inactive/Closed, but not with negative allocation or negative allowable percentages. (acq-models `budget.json`; UIF-573, MODFIN-405; C663288)
32. **Batch allocation**: limited to 1000 funds per ledger and available only from Ledger Actions -> Allocation tools; new budgets get `Active` (current FY) / `Planned` (future FY) unless a valid status is set; increasing allocation on an Inactive/Frozen fund makes it Active; `0` is valid (budget without allocation transaction), a negative total is rejected; if any budget is not Active/Planned the whole update fails. Details, messages and sources: Batch Allocation section.
33. **Rollover specifics** (partial encumbrance-type selection, `Based on` / `Rollover budget value` / `Rollover value as` options, rounding, `Net transfers` after Recalculate, multi-ledger POLs, logs, removed expenditure-limit toggle, unrelease blocked when payments exist): see Exact UI Texts → Rollover page and `Rollover - additional rules`.

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
- Fund/budget `Unassigned` expense class row shows totals for transactions without an expense class (UIF-617, MODFIN-433, en.json `budget.expenseClasses.unassigned.tooltip`: `"Unassigned" applies to transactions completed prior to the inclusion of expense classes on the budget, if there are any that exist.`; C15857 area; Cypress `totals-for-transactions-without-expense-class-are-included-in-expense-class-summary.cy.js`). Percent of total expended must stay correct when over-expended (MODFIN-372, C496115, C440081).
- Over-encumbrance is calculated per budget/ledger/FY (MODFIN-373, MODFISTO-484, C496175, C496145/149/153).
- Budget validation on update (MODFIN-374; C503053): remaining-allowed-encumbrance / expenditure checks take paid and pending credits into account (C496166, C496167).
- Fiscal year currency: see rule 2. Period End Date is end of the selected day, UTC (UIF-666); FY form/view shows `UTC` (Cypress `utc-is-displayed-when-create-or-view-fiscal-year.cy.js`).
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
- Fiscal year summary does not show Net transfers (transfers only within one FY); Net transfers appear on Ledger, Group and Budget summaries.

---

## Known Gaps / Items to Verify

- [ ] Jira scope: no fixVersion exists for Umbrellaleaf R2 2026 on UIF/MODFIN/MODFISTO yet (latest seen: UIF 9.0.0, MODFIN 5.2.0, MODFISTO 8.9.0 = Trillium R1 2026; master is UIF 9.1.0 / MODFIN 6.0.0-SNAPSHOT); items after 2026-04 were scoped by resolution date (UIF-544, UIF-666, MODFIN-465, MODFISTO-565).
- [ ] Browse tab (UIF-544..548): commits showed revert and re-introduction in Aug–Oct 2026; settings page and `browse.*` strings exist in en.json; only a DRAFT TestRail case (C788678) — verify in target env before asserting.
- [ ] Rollover confirm dialog text differs between `en.json` (current) and `en_US.json`/old file; this file now uses en.json. The `FOLIO has found invoices that are not yet paid...` unpaid-invoice message is unchanged.
- [ ] Docs (docs.folio.org) list Frozen/Inactive budget statuses and say a budget can be "Frozen"; older TestRail cases mention only Active/Planned/Closed — expected behavior of Frozen/Inactive budgets for transactions is not documented in any source read.
- [ ] Rules 'Deleting Records' table (fund/ledger/group/FY delete conditions) and 'Over encumbered/over expended only shows if ledger restrictions disabled' have no source in this refresh; keep conservative. 2026-11-25 pass: docs give only the general rule (records with assigned records cannot be deleted; budgets with non-allocation transactions cannot be deleted; funds in a rollover cannot be deleted) - per-record conditions beyond that are marked (unverified) in the `Deleting Records` table.
- [ ] `en.json` exposes `transaction.transfer.budgetRestrictedExpendituresError` with different wording (`{amount} could not be transferred to the budget <b>{toBudgetName}</b> because it exceeds the allowable expenditure limit for <b>{fromBudgetName}</b> and ledger fund restrictions are active.`) than the `states.js`-sourced string in the Error table; check which one the UI renders per case.
- [ ] Confluence (folio-org.atlassian.net/wiki anonymous REST) returned only Acquisitions SIG meeting notes / RM SIG pages for "fiscal year rollover"; no Finance requirement page was found or relied on.
- [ ] Jira Done search returned no entries with fixVersion naming Umbrellaleaf/Trillium/Sunflower/Ramsons by release name (fixVersions are module versions); mapping used: Ramsons R2 2024 = UIF 7.0.0 / MODFIN 5.0.0 / MODFISTO 8.7.0; Sunflower R1 2025 = UIF 8.0.0 / MODFIN 5.1.0 / MODFISTO 8.8.0; Trillium R1 2026 = UIF 9.0.0 / MODFIN 5.2.0 / MODFISTO 8.9.0 (from "Release" tasks UIF-558/599/660, MODFIN-383/420/459, MODFISTO-505/521/556).
- [ ] Batch allocation worksheet column list beyond the ones confirmed in C648502 was not re-verified against the BE schema (`finance-data.json` not read).
- [ ] Fund `Ledger` "cannot be changed after creation" (old Fund rule): docs say only that a fund has one ledger; `ui-finance` `FundForm.js` does not disable the Ledger select in edit mode and no BE check was found in mod-finance (`FundsHelper`) - reassignment behaviour (UI/API) unverified.
- [ ] Behaviour of Frozen/Inactive **ledgers, funds** (not only budgets) for transactions: docs only say "paused" / "no longer in use"; the earlier statement "Frozen - no new transactions" had no source and was reworded.
- [ ] Orders/Invoices-side claims with no Finance source read: restricted-unit funds hidden from fund-distribution dropdowns; a bypass-acquisition-units capability.
- [ ] Capability sets: `Manage acquisition units` - docs say `Data - UI-Finance Acq Unit Assignment - Manage`, package.json id is `...acquisitions_units.manage.execute` (Procedural/Execute shape), Cypress only has the Assign set (`UI-Finance Finance Acquisitions Units Assign`, procedural execute; docs spell it `Acq Unit Assignment`) - exact names marked `(capability set name not confirmed — verify)`. Ledger/Group/Fund-Budget Edit/Create/Delete, Rollover, Release/Unrelease, Recalculate, Fund-Update-Logs and Settings Manage/ExportFundAndExpenseClassCodes/Exchange Rate Source names come from docs.folio.org only (not in the Cypress dictionary); verify in the target env.
- [ ] Docs list `Credited` as part of `Unavailable` (Unavailable = encumbered + awaiting payment + credited + expended); the remaining-allowed-encumbrance formula in rule 26 (C449364) omits credited - verify with a credit-invoice case (C496166/C496167).
- [ ] Transactions: mod-finance-storage has no PUT/DELETE for transactions, but release/unrelease/encumbrance updates are applied via `batch-all-or-nothing`; the old statement "transactions cannot be edited or deleted" was narrowed to the API fact.
- [ ] TestRail was not re-read in the 2026-11-25 pass (size/traceability pass only); no new case bodies were read.
- [x] "Exact UI Texts" seeded 2026-07-21 and refreshed against en.json 2026-11-24 (batch allocation, upload validation, logs, exchange rate, navigation, rollover options added).
- [x] The old note that Jira returned 0 Done issues for component "Finance" is obsolete: project keys UIF / MODFIN / MODFISTO are used (93 / 63 / 69 Done issues resolved since 2024-06-01).

> Random spot-check (2026-07-22): picked one fresh uncited case at random from the "Fiscal Year Rollover" subsection (C376609) — a large, detailed rollover scenario. The general "Planned budget is created" rule already existed (21) but didn't cover the specific zeroed-out outcome when Rollover allocation is off and Rollover budget value is None. Added Rule 30 as a refinement rather than a new top-level fact.


---

## Changes since 2026-07-21 (delta refresh 2026-11-07)

> Delta, not a rebuild: sections above reflect the original build. Where this section contradicts them, **this section wins**.
> Sources: GitHub folio-org/ui-finance (translations/ui-finance/en.json, HEAD v9.0.0); Jira folio-org.atlassian.net Done issues resolved since 2026-07-21; TestRail suite 21 cases created since 2026-07-21.

### What changed
- **Browse** tab in Finance (Search | Browse): select a fiscal year, hierarchy Ledger → Group → Fund → Budget → Expense class, status filters (Ledger/Group/Fund/Budget/Expense class status), New fiscal year/ledger/group/fund actions (UIF-545..548; enabled via finance navigation setting UIF-544). Commits show a revert on 2026-08-18 and re-introduction in Sep–Oct — verify the tab is present in the target env before asserting.

### UI strings changed since 2026-07-21 [source — en.json, 2026-11-07]

> `en.json` is the source of truth; `en_US.json` is a Lokalise copy that can lag behind. `<b>`/`<strong>` render as bold; `{x}` are placeholders.

**New strings** (15 keys, all under `browse.*`): `browse.tab.search` = Search; `browse.tab.browse` = Browse; `browse.title` = Browse finance; `browse.subtitle` = Select a fiscal year to browse results; `browse.selectFiscalYear` = Select a fiscal year to browse results.; `browse.filter.fiscalYear` = Fiscal years; `browse.filter.ledgerStatus` = Ledger status; `browse.filter.groupStatus` = Group status; `browse.filter.fundStatus` = Fund status; `browse.filter.budgetStatus` = Budget status; `browse.filter.expenseClassStatus` = Expense class status; `browse.actions.newFiscalYear` = New fiscal year; `browse.actions.newLedger` = New ledger; `browse.actions.newGroup` = New group; `browse.actions.newFund` = New fund

### Jira — Done since build (finance (UIF, MODFIN) since 2026-07-21)

- UIF-666 Bug — Fiscal year Period End Date displays as end of the selected day (23:59 UTC); see Display rules
- (GitHub only, no Done ticket yet) UIF-544..548 Browse tab / finance hierarchy

### TestRail — cases created since 2026-07-21: 1 (1 manual, 0 automated/Karate)

- C1504439 Create and update budget via upload allocation worksheet (CSV) for the Current fiscal year from Ledger view — refs UIF-526,UIF-527,UIF-528,UIF-529,UIF-591,MODFIN-419,UIF-628,UISACQCOMP-283,MODFIN-443,MODFIN-442,MODFIN-444
