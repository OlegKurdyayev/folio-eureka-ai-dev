# FOLIO Fees/Fines — Business Logic Context

> Built: 2026-11-25 · Release: Umbrellaleaf R2 2026 (master: ui-users 13.1.0 in progress, mod-feesfines 19.5.0-SNAPSHOT; release mapping from CHANGELOG dates, see Known Gaps) · Builder: build-app-context (refresh of the 2026-07-21 build + 2026-11-07 delta)
> Sources: TestRail project 14 / suite 21, section 88 "Fees&Fines" and subsections (Settings Users (Fee/fine) 89, Manual 90, Pay 91, Display 92, Waive 1533, Transfer 1534, Cancel 1530, Refund 1532, Overdue Fines 1141, Reminder Fees 25512, Declared Lost 1312, Aged to Lost 1531, Claimed Returned 1313, Bursar 11060, Financial Transactions Detail Report 16509, Cash Drawer Reconciliation Report 16510, Lost items requiring actual cost 20624). Second pass also read case bodies from Declared Lost 1312, Aged to Lost 1531, Overdue Fines 1141, Permissions --> Fees and Fines 183, Bursar 11060.
>          GitHub: `folio-org/ui-users` (translations/ui-users/en.json, package.json `stripes.permissionSets`, src/components/Accounts/**), `folio-org/mod-feesfines` (descriptors/ModuleDescriptor-template.json, ramls/, src/main/java/.../service/action/**, NEWS.md), `folio-org/folio-integration-tests` (mod-feesfines), `folio-org/stripes-testing`, `folio-org/docs` (content/en/docs/Access/Additional topics/feesfines/feesfines.md).
>          Jira: UIU + MODFEE Story/Bug, Done/Closed, resolved ≥ 2024-06-01 (fixVersions hold module versions, not release names).
> UI strings: `en.json` key named in brackets = exact string from source; `C…` = string asserted in a TestRail case. Where this file and an older case disagree, the `en.json` key wins.

---

## What is Fees/Fines

Fees/Fines are monetary charges against a patron — either **manual** (staff-created via a configured Manual charge type, e.g. replacement card, damage) or **automatic** (created by circulation per policy: Overdue fine, Lost item fee, Lost item processing fee, Reminder fee). Staff manage them from the patron's **Fees/fines** accordion in the Users app: charge, pay, waive, transfer, refund, cancel as error, add staff info, export. Settings live under `Settings > Users` (Fee/fine group); overdue / lost-item / reminder-fee policies live under `Settings > Circulation`. Manual charges are always linked to a patron; they may be linked to an item but never to a loan (docs feesfines.md).

---

## Key Terms

| Term | Definition |
|---|---|
| Account | The fee/fine record (`/accounts`). Has `amount` (billed), `remaining`, `status.name` (Open/Closed), `paymentStatus.name`. Parent object (docs feesfines.md; ramls/accountdata.json). |
| Fee/fine action | Child transaction of an account (`/feefineactions`): charge, payment, waive, transfer, refund, credit, cancel, staff info. One account → 1..n actions (ramls/feefineactiondata.json: required `accountId`, `userId`). |
| Fee/fine owner | Office/agent that owns charges; manual charges, payment methods, transfer accounts are per owner. Owners are linked to service points; automatic fees pick the owner via item location → primary service point → owner (docs feesfines.md). |
| Shared owner | An owner named `Shared`; cannot have associated service points (C350615, en.json `owners.noServicePoints`). |
| Manual charge (Fee/fine type) | Owner-scoped charge type with default amount and charge/action notices. Names of automated types are reserved (`feefines.errors.reservedName`). |
| Payment method | Owner-scoped (`ownerId`); has `Refund method allowed` flag (`allowedRefundMethod`) (ramls/paymentdata.json; en.json). |
| Transfer account | Owner-scoped (`ownerId`) external account (e.g. bursar) a balance can be transferred to (ramls/transferdata.json). |
| Waive reason / Refund reason | Tenant-wide reason lists, no `ownerId` field (ramls/waivedata.json, refunddata.json). |
| Suspended | Lost-item fees on a Claimed-returned loan: `paymentStatus` = `Suspended claim returned`, blocked from actions; counted in "Total suspended balance" (ramls/accountdata.json `paymentStatus`; en.json `accounts.history.*`; UIU-3125). |
| Credit | Internal action created before every Refund (paired Credit + Refund) (RefundActionService.java; C15209). |

---

## Setup / Settings

`Settings > Users` (Fee/fine group, en.json `settings.*`): **Owners**, **Manual charges**, **Waive reasons**, **Payment methods**, **Refund reasons**, **Comment required**, **Transfer accounts**, **Transfer criteria** (bursar export plugin; shows `No plugin available!` without it). Plus `Settings > Circulation > Fee/fine`: Overdue fine policies (incl. Reminder fees), Lost item fee policies, Fee/fine notice triggering events.

| Page (en.json) | Columns / fields (en.json) | Validations / guards |
|---|---|---|
| `Fee/fine: Owners` (`owners.label`) | `Owner`, `Description`, `Associated service points` | `This Fee/fine Owner cannot be deleted, as it has one or more fees/fines associated with it`; `Associated service points not allowed for Shared fee/fine owner`; warning `Warning: Overdue fines/lost item fees will not be collected without a service point`; empty `There are no Fee/fine: Owners` |
| `Fee/fine: Manual charges` (`feefines.title`) | owner select `Owner`; `Fee/fine type`, `Default amount`, `Charge notice`, `Action notice`; `Default charge notice` / `Default action notice` | `Default Amount must be numeric`, `Default Amount must be positive`, `Fee/fine type exists`, `Fee/fine type exists for other Fee/fine owner(s)`, `Fee/fine type exists for Shared Fee/fine owner`, `Fee/fine type already exists as automated fee/fine`; delete modal `Delete Fee/Fine Type?` / `Fee/fine Type cannot be deleted when used by one or more user accounts`; copy modal `Copy existing fee/fine owner table entries?` (`Select one to continue`) |
| `Fee/fine: Payment methods` | `Name`, `Refund method allowed` | empty `There are no Fee/fine: Payment methods` |
| `Fee/fine: Waive reasons` | `Reason`, `Description` | — |
| `Fee/fine: Refund reasons` | `Name`, `Description` | — |
| `Fee/fine: Transfer accounts` | `Name`, `Description` | delete modal `Delete transfer account`; empty `There are no Fee/fine: Transfer accounts` |
| `Fee/fine: Comment required` | `Require comment when fee/fine fully/partially paid` / `…waived` / `…transferred` / `…refunded`; `Save` → `Comment required settings were successfully updated.` | — |

Owner fallback: an owner should be configured for every service point, otherwise automatic fee creation fails (docs feesfines.md). When no owner is found for the item's primary service point, the UI shows `Lost item fee(s) not billed to patron` (`feefines.errors.notBilledTitle` / `notBilledMessage`).

---

## Charge lifecycle and statuses

```
(sources: ramls/accountdata.json, mod-feesfines service/action/*, FeeFineActions.js; rules 6, 10-13, 22)
(Manual)    "New fee/fine" → "Charge only"  |  "Charge & pay now" (→ Pay modal)
(Automatic) circulation creates Overdue fine / Lost item fee / Lost item processing fee / Reminder fee

Open (remaining > 0)
  ├─ Pay / Waive / Transfer  (partial → stays Open; full → Closed)
  ├─ Error (cancel as error) → Closed, "Cancelled as error"   [only while nothing was paid/waived/transferred]
  └─ Claimed returned (lost-item fees) → "Suspended claim returned", actions blocked
Closed (remaining = 0)
  └─ Refund (only of paid/transferred amount) → Credit + Refund actions; account status set back to Open
```

- `status.name`: `Open`, `Closed` (ramls/accountdata.json).
- `paymentStatus.name` enum (ramls/accountdata.json; UI labels `accounts.paymentStatus.*`): `Outstanding`, `Paid partially`, `Paid fully`, `Waived partially`, `Waived fully`, `Transferred partially`, `Transferred fully`, `Refunded partially`, `Refunded fully`, `Credited fully`, `Credited partially`, `Cancelled item returned`, `Cancelled item renewed`, `Cancelled item declared lost`, `Cancelled as error`, `Suspended claim returned`.
- Action types written to history: the same "… partially/… fully" strings, plus `Staff info only` (new staff info), `Cancelled as error` (mod-feesfines Action.java / service/action/*).
- Refund transaction info: Credit → `Refund to patron` / `Refund to {transfer account}`; Refund → `Refunded to patron` / `Refunded to {transfer account}` (RefundActionService.java).
- When all fees of a lost-item loan are resolved by pay/waive/cancel, FOLIO closes the loan and sets item status `Lost and paid` (docs feesfines.md); if returned and all fees removed → loan closed, item Available / In transit.

---

## UI structure

- **Patron record → `Fees/fines` accordion** (`accounts.title.feeFine`): counts `{n} open fees/fines` + `(Total: {amount})`, `{n} closed fees/fines`, `{n} suspended claim returned fees/fines`, `{n} refunded fees/fines`; links `View all fees/fines`; button `New fee/fine` (`accounts.chargeManual`; C808511). The same `New fee/fine` option is in the Loans ellipsis and Check in ellipsis menus (UIU-3602, UIU-3193).
- **Fees/fines history page** (AccountsListing.js; IDs from `userAllFeesFines.js` fragment):
  - Header `Fees/fines - {username}`; tabs `Open` (`open-accounts`), `Closed` (`closed-accounts`), `All` (`all-accounts`); list id `list-accounts-history-view-feesfines`.
  - Totals: `Total outstanding balance: {amount}`, `Total suspended balance: {amount}`, `Outstanding balance for page: {amount}`, `Selected fees/fines balance: {amount}`.
  - Columns (`accounts.history.columns.*`): `Date created`, `Date updated`, `Fee/fine type`, `Billed`, `Remaining`, `Payment status`, `Fee/fine owner`, `Instance (Material type)`, `Barcode`, `Effective call number string`, `Due date`, `Returned date`, `Actions` (header added by UIU-3027); `Select columns`.
  - Page buttons: `New fee/fine`, `Pay`, `Waive`, `Transfer`, `Refund`, `Export fees/fines (CSV)`.
  - Row ellipsis: `Pay`, `Waive`, `Transfer`, `Error` (cancel as error, `accounts.button.error`), `Refund`, `Loan details` / `Loan details (anonymized)`, `Go to details`.
  - Action buttons are hidden (not disabled) when the user lacks the permission (UIU-3156 in ui-users CHANGELOG).
- **Fee/fine details page** (`Fee/fine details`; pane id `pane-account-action-history`): header `Fee/fine details - {userName} ({patronGroup})`; fields `Fee/fine type`, `Fee/fine owner`, `Billed date`, `Billed amount`, `Remaining amount`, `Latest payment status`, `Instance (Material type)`, `Item type`, `Barcode` (`{barcode} (not found)` if item missing — C358152), `Effective call number string`, `Location`, `Due date`, `Returned date`, `Loan details` (`View`), plus links `Barcode` (Item details), `Overdue policy`, `Lost item policy` that must open the right records (C11149 — recurring Barcode-link problem). Buttons: Pay / Waive / Refund / Transfer / `Error` / Export (ids `payAccountActionsHistory`, `waiveAccountActionsHistory`, `refundAccountActionsHistory`, `transferAccountActionsHistory`, `errorAccountActionsHistory`, `exportAccountActionsHistoryReport`), `New staff info`.
  - Action history columns (`details.columns.*`): `Action date`, `Action`, `Amount`, `Balance`, `Transaction information`, `Created at`, `Source`, `Additional information`. Staff/patron comment prefixes `STAFF : {text}` / `PATRON : {text}` (`accounts.actions.tag.*`, FeeFineActions.js `assembleTagInfo`).
  - Page must survive reload and direct-URL open (UIU-3428, C808511).
  - Export (CSV) of selected fees/fines: one row per action; columns `Patron name`, `Patron barcode`, `Patron group`, `Action date`, `Action description`, `Action amount`, `Action balance`, `Action transaction information`, `Action created at`, `Action source`, `Action additional information for staff` / `…for patron`, `Fee/fine type`, `Fee/fine owner`, `Fee/fine billed date` / `billed amount` / `remaining amount` / `latest payment status` / `details`, `Item instance`, `Item material type`, `Item barcode`, `Item call number`, `Item location`, `Item due date`, `Item returned date`, `Item overdue policy`, `Item lost item policy`, `Item loan details` (C359591; UIU-1605).
- **New fee/fine page** (`New fee/fine`, ChargeForm.js; modal id `new-modal`): `Fee/fine owner` (`ownerId`), `Fee/fine type` (`feeFineType`), `Fee/fine amount` (`amount`), `Additional information for staff` (`comments`), `Notify patron` (`notify`), `Item information` block (`Scan or enter item barcode`, `Enter`, `Barcode`, `Instance`, `Item Status`, `Effective call number string`, `Location`). Buttons `Cancel` (`cancelCharge`), `Charge & pay now` (`chargeAndPay`, en.json `charge.Pay`), `Charge only` (`chargeOnly`).
  - Unknown barcode: `No item with barcode {barcode} could be found`, both charge buttons disabled (UIU-2701).
  - No current service point: modal `Access denied` — `You must select a service point to create and view patron fees/fines. …` (`errorModal.currentUserServicePointAbsentee.*`; button `Back`).

---

## Exact UI Texts

### Callouts (toasts)

| Event | Text (key) | Source |
|---|---|---|
| Pay / Waive / Transfer / Refund (single and bulk), Cancel as error | `{count} fee/fine for {amount} has been successfully {action} for {user}` (`accounts.actions.cancellation.success`); `{action}` = `fully paid` / `partially paid` / `fully waived` / `partially waived` / `fully transferred` / … / `cancelled as error` | FeeFineActions.js `showCalloutMessage`; Cypress `cancelFeeFine.js` asserts `has been successfully cancelled as error for ` |
| Manual charge created | `The {feeFineType} fee/fine of {amount} has been successfully {paymentName} for {fullName}` (`charge.message.successfully`; `{paymentName}` = lowercased payment status — for Charge & pay now `paid fully` / `paid partially`) | ChargeFeeFine.js |
| Actual cost billed | `A fee/fine of {amount} has been successfully charged to {patronName}` (`lostItems.notification.billed`) | en.json; Cypress `bill-actual-cost-…cy.js` |
| Actual cost not billed | `A lost item fee will not be charged to {patronName}` (`lostItems.notification.cancelled`) | en.json |
| Manual charge type deleted | `The Fee/Fine Type {feefine} was successfully deleted.` (`feefines.callout`) | en.json |

> Callout quirks (FeeFineActions.js): the `{amount}` shown is the account's **billed** `amount`, not the action amount; in bulk actions one callout per selected account with `count` = 1. `accounts.actions.calloutMessage` (`The {feeFineType} fee/fine of {amount} has been successfully {paymentStatus} for {name}.`) exists in en.json but is **not referenced in src** — do not assert it.

### Action modals

| Action | Modal title | Amount field | Method field | Summary (full / partial) |
|---|---|---|---|---|
| Pay | `Pay fee/fine` | `Payment amount` | `Payment method` (`Select type`) | `Paying` / `Partially paying` |
| Waive | `Waive fee/fine` | `Waive amount` | `Waive reason` (`Select reason`) | `Waiving` / `Partially waive` |
| Transfer | `Transfer fee/fine` | `Transfer amount` | `Transfer Account` (`Select account`) | `Transferring` / `Partially transfer` |
| Refund | `Refund fee/fine` | `Refund amount` | `Refund reason` (`Select reason`) | `Refunding` / `Partially refunding` |

- Summary line: `{verb} {count} fee/fine(s) for a total amount of {amount}` (`accounts.summary`; Cypress `waiveFeeFineModal.js`, C15190).
- Common fields: `Fee/fine owner desk` (`Select desk`), `Transaction information` (`Enter check #, etc.`, Pay only), `Additional information for staff`, `Notify patron`, `Additional information for patron`, `Total owed amount`, `Selected amount`, `Remaining amount`; Refund shows `Total paid amount` (UIU-3094 fixed `$NaN`).
- Amount inputs are number fields (UIU-2836): leading spaces/letters cannot be typed; comma decimal not accepted (UIU-3206, Won't Do).
- Page/menu labels: `Pay fees/fines`, `Waive fees/fines`, `Transfer fees/fines`, `Refund fees/fines`.

### Confirmation step (every Pay / Waive / Transfer / Refund)

- Title `Confirm fee/fine {action}` (`accounts.confirmation.head`) with `{action}` = `payment` / `waive` / `transfer` / `refund` (`accounts.actions.*`) → **`Confirm fee/fine payment`**, **`Confirm fee/fine waive`**, **`Confirm fee/fine transfer`**, **`Confirm fee/fine refund`** (FeeFineActions.js `renderConfirmHeading`; Refund title also in Cypress `refundFeeFine.js`). "Charge & pay now" also uses `Confirm fee/fine payment` (ChargeFeeFine.js).
- Body `{count} fee/fine for {amount} will be {fully|partially} {paid|waived|transferred|refunded}` (`accounts.confirmation.message`). Full vs partial = entered amount vs remaining (refund: vs billed amount for single, vs sum of `remaining` for bulk).
- Cancel as error: one modal `Confirm fee/fine cancellation` (modal id `error-modal`; buttons `Back` / `Confirm`), body `{amount} {feeFineType} fee/fine will be cancelled`; `Confirm` needs a comment (`Comment must be provided`, `accounts.cancellation.error.comment`; Cypress `cancelFeeFine.js`).

### Validation / error messages

| Condition | UI message (key) | API message (mod-feesfines) |
|---|---|---|
| Pay ≤ 0 / > selected | `Payment must be > 0` / `Pay amount exceeds the selected amount` | `Amount must be positive` / `Requested amount exceeds remaining amount` |
| Waive ≤ 0 / > selected | `Waive amount must be > 0` / `Waive amount exceeds the selected amount` | same as above |
| Transfer ≤ 0 / > selected | `Transfer amount must be greater than zero` / `Transfer amount exceeds selected amount` | same as above |
| Refund > refundable | — | `Refund amount must be greater than zero and less than or equal to Selected amount` |
| Non-numeric amount | — | `Invalid amount entered` |
| Action on a closed fee/fine (pay/waive/transfer/cancel) | control hidden/disabled | `Fee/fine is already closed` |
| Unknown account | — | `Fee/fine ID {id} not found` |
| No payment methods / waive reasons / transfer accounts | `No Payment methods exist for Fee/fine owner` / `No Waive reasons exist` / `No Transfer accounts exist for Fee/fine owner` | — |
| No refund reason | `Refund reason is required` (`accounts.refund.error.select`) | — |
| Comment required (owner setting) | `Additional information for staff is required` (`accounts.error.comment`) | — |
| Required field | `Please fill this field in to continue` (`accounts.error.field`) | — |
| Mixed selection | Modal `Alert details` + `Deselect to continue`; text `{n} items selected. {m} item cannot be {action} because it is already closed.` / `… because it was not paid.` (refund) (`accounts.warning.itemSelected` / `unpaidItemSelected`) | — |

API messages: DefaultActionValidationService / RefundActionValidationService / CancelActionValidationService. The older note that `Amount must be positive` / `Requested amount exceeds remaining amount` were "2020 mock text" was wrong — they are the real API `errorMessage` values; the UI shows the en.json strings instead.

### Reports (Users app > Actions)

| Report (menu label) | Permission | Validation |
|---|---|---|
| `Financial transaction detail report (CSV)` → modal `Financial transaction detail report` | `Users: Create and download Financial transaction detail report` | `"Start date" is required`, `"Start date" is required if "End date" entered`, `"End date" must be greater than or equal to "Start date"`, `"Fee/fine owner" is required` (C343207, C343311, C343314, C343324) |
| `Cash drawer reconciliation report (CSV, PDF)` → modal `Cash drawer reconciliation report` | `Users: Create and download Cash drawer reconciliation report` | same date rules + `"Service point" is required` |
| `Refunds to process manually report (CSV)` → modal `Refunds to process manually` (`Specify date refund entered range`) | `Users: Create and download Refunds to process manually report` | `Must enter Start date if End date provided`, `End date must be after Start date` (C196745) |

- Headers: `Financial Transactions Detail Report for {owner}, Service Point(s) {servicePoints} - {startDate} to {endDate}`; `Cash Drawer Reconciliation Report for {servicePoint}, Source(s) {sources} - {startDate} to {endDate}` (en.json `reports.*.header`).
- Cash drawer: selecting a Service point fills `Sources` with users who acted at that service point; defaults to the logged-in user if associated (cases C343xxx). Totals sections: `Fee/fine type totals`, `Fee/fine owner totals`, `Payment method totals` (+ Source/Waive/Refund/Transfer totals in API, mod-feesfines report builders).
- API: `POST /feefine-reports/financial-transactions-detail`, `/cash-drawer-reconciliation`, `/cash-drawer-reconciliation/sources`, `/refund`; refund report → 422 on invalid params, empty when refund after end date, valid with null dates (Karate feeFineReports.feature).

---

## API surface (mod-feesfines `feesfines` 19.1, `feefine-reports` 2.3, `actual-cost-fee-fine` 0.3; descriptors/ModuleDescriptor-template.json, verified 2026-11-25)

| Endpoint | permissionsRequired |
|---|---|
| `GET/POST /accounts`, `GET/PUT/DELETE /accounts/{id}` | `accounts.collection.get`, `accounts.item.post/get/put/delete` |
| `POST /accounts/{id}/check-pay` · `check-waive` · `check-transfer` · `check-refund` | `feesfines.accounts.check-{pay,waive,transfer,refund}.post` |
| `POST /accounts/{id}/pay` · `waive` · `transfer` · `cancel` · `refund` | `feesfines.accounts.{pay,waive,transfer,cancel,refund}.post` |
| `POST /accounts-bulk/check-{pay,waive,transfer,refund}` | `feesfines.accounts-bulk.check-*.post` |
| `POST /accounts-bulk/{pay,waive,transfer,cancel,refund}` | `feesfines.accounts-bulk.*.post` |
| `/feefineactions`, `/feefines` (manual charge types), `/owners`, `/payments`, `/waives`, `/refunds`, `/transfers`, `/comments`, `/transfer-criterias`, `/manualblocks`, `/manual-block-templates`, `/overdue-fines-policies`, `/lost-item-fees-policies` (CRUD) | `<resource>.collection.get` / `<resource>.item.{get,post,put,delete}` |
| `POST /actual-cost-fee-fine/bill` · `/cancel` | `actual-cost-fee-fine-bill.post` / `actual-cost-fee-fine-cancel.post` |
| `POST /feefine-reports/refund` · `/financial-transactions-detail` · `/cash-drawer-reconciliation` · `/cash-drawer-reconciliation/sources` | `feefine-reports.refund.post`, `feefine-reports.financial-transactions-detail.post`, `feesfines.feefine-reports.cash-drawer-reconciliation.post`, `feesfines.feefine-reports.cash-drawer-reconciliation-source.post` |

- Account required fields: `amount`, `remaining`, `status`, `paymentStatus`, `userId`, `feeFineId`, `ownerId` (ramls/accountdata.json); POST `/accounts` generates an id if missing (MODFEE-296).
- Check-* response: `accountId`, `amount`, `allowed`, `remainingAmount`, `errorMessage` (ramls/actions/checkActionResponse.json). Action endpoints: 404 unknown account, 422 for zero/negative/invalid amount or closed account (Karate accounts.feature).
- Policy names are unique: `The Overdue fine policy name entered already exists. Please enter a different name.` / `The Lost item fee policy name entered already exists. Please enter a different name.` (mod-feesfines Java).
- Fees/fines are exposed to Lists/FQM as entity types (MODFQMMGR-905/1066, NEWS 19.4.0); Cypress specs under `cypress/e2e/lists/query-builder/**fees-fines**`.

---

## Capability Sets (Eureka) and legacy permissions

| Action | Capability set (type - resource - action) | Legacy permission (ui-users package.json) |
|---|---|---|
| View fees/fines, details, history | `Data - UI-Users Feesfines - View` (Cypress dictionary `uiUsersFeeFinesView`; C808511, C491280) | `Users: Can view fees/fines and loans` (`ui-users.feesfines.view`) |
| Charge, pay, waive, transfer, refund, cancel (all actions) | `Data - UI-Users Feesfines Actions - Manage` (C375239, C491280; UIU-3602). Not in the Cypress dictionary — its `uiUsersFeeFineActionManage` is `Data - UI-Users Fee-Fine-Actions - Manage`, a different set | `Users: Can create, edit and remove fees/fines` (`ui-users.feesfines.actions.all`) |
| Pay only | `Procedural - UI-Users Manual-Pay - Execute` (Cypress dictionary `uiUsersManualPay`; C375239) | `Fees/Fines: Can pay` (`ui-users.manual-pay.execute`) |
| Waive only | `Procedural - UI-Users Manual-Waive - Execute` (capability set name not confirmed — verify) | `Fees/Fines: Can waive` (`ui-users.manual-waive.execute`) |
| Charge (incl. pay) | `Procedural - UI-Users Manual-Charge - Execute` (capability set name not confirmed — verify) | `Fees/Fines: Can charge` (`ui-users.manual-charge.execute`; package.json includes `ui-users.manual-pay.execute`) |
| Fee/fine actions history CRUD | `Data - UI-Users Fee-Fine-Actions - Manage` (Cypress dictionary `uiUsersFeeFineActionManage`; mapping to the legacy permission by naming only) | `Fee/Fine Details: Can create, edit and remove fee/fine actions` (`ui-users.fee-fine-actions.all`; included in Can pay / Can waive) |
| Reports | (capability set name not confirmed — verify) | `Users: Create and download Financial transaction detail report` / `… Cash drawer reconciliation report` / `… Refunds to process manually report` (package.json) |
| Actual cost processing | `Procedural - UI-Users Lost-Items-Requiring-Actual-Cost - Execute` (C359236 preconditions) | `Users: Can process lost items requiring actual cost` (`ui-users.lost-items-requiring-actual-cost.execute`) |
| Settings — all fee/fine settings | (capability set name not confirmed — verify); legacy set `ui-users.settings.feefines.all` | `Settings (Users): Can create, edit and remove all feefines-related entries`; per-page: `… Can create, edit and remove owners` / `manual charges` / `payment methods` / `refunds reasons` / `waive reasons` / `transfer accounts`; view-only `Settings (Users): Can view feefines-related entries` (package.json) |

- In package.json `Users: Can create, edit and remove fees/fines` (`ui-users.feesfines.actions.all`) already includes `ui-users.feesfines.view`, `manual-charge`, `manual-pay` and `manual-waive`, plus the transfer / refund / cancel API permissions; `Fees/Fines: Can pay` and `Can waive` each pull in `ui-users.fee-fine-actions.all`.
- There is **no separate refund or transfer permission** — refund, transfer and cancel come only from `Users: Can create, edit and remove fees/fines` (package.json). The former `Procedural - UI-Users Feefine Refund - Execute` / `Data - UI-Users Feesfines - Create/Edit` names had no source and are removed.
- `Fees/Fines: Can pay` alone allows Pay (Export disabled); `Fees/Fines: Can waive` alone allows Waive (UIU-3109; C494013, C494014).
- `Settings (Users): View all settings` user can view but not edit Manual charges, Transfer accounts, Comment required (C407702–C407704, C411867).

---

## ECS / Multi-Tenant Notes

Fees/fines are tenant-local (owners, accounts, policies per tenant); no ECS-only test cases found in section 88. Bugs are routinely re-verified on both ECS and non-ECS snapshots (UIU-3549, UIU-3602). DCB (OpenRS) service point as fee/fine owner can create repeated lost-item bills when the item does not move to Aged to lost — closed Cannot Reproduce (MODFEE-450).

---

## Coverage already in place

- **Karate** (`folio-integration-tests/mod-feesfines/src/main/resources/vega/mod-feesfines/features/`, runner `feesfines-junit.feature`): `accounts.feature` (53 scenarios: CRUD, check-pay/waive/transfer/refund amount rules, closed-account rules, bulk check/refund, 404/422 action errors, long decimals), `manualFeeFines.feature` (pay immediately / separately, transfer, waive, cancel, refund to patron / to transfer account), `feeFineActions.feature`, `feeFineTypes.feature`, `owners.feature`, `feeFineReports.feature` (refund report), `actualCostFeeFine.feature`, `agedToLostFeeFine.feature`, `automateFeeFines.feature`, `overdueFinePolicies.feature`, `lostItemFeePolicies.feature`, `manualBlocks.feature`, `manualBlockTemplates.feature`. Not covered by Karate: cash-drawer and financial-transactions reports, transfer-criteria, comments.
- **Cypress** (`stripes-testing`): specs `cypress/e2e/fees-&-fines/**` (pay, waive, transfer, refund, cancel/error, details reload, notices, actual cost, financial-transactions/**, patron blocks), `cypress/e2e/settings/users/fees&fines/**` (owners, manual charges, payment methods, waive/refund reasons, comment required, view-only permissions); fragments `cypress/support/fragments/users/{payFeeFine,waiveFeeFineModal,transferFeeFine,refundFeeFine,cancelFeeFine,feeFineDetails,userAllFeesFines,newFeeFine,financialTransactionDetailReportModal}.js`, `fragments/settings/users/{usersOwners,paymentMethods,refundReasons,transferAccounts,waiveReasons,transferCriteria}.js`.

---

## Common Verification Patterns

### Charge a manual fee/fine
Preconditions: Owner + Manual charge (+ Payment method for "Charge & pay now"); active patron; staff user has a service point.
1. Patron record → `Fees/fines` → `New fee/fine`.
2. Select `Fee/fine owner`, `Fee/fine type`, amount → `Charge only`.
3. Expected: success callout (`charge.message.successfully`); row in `Open` tab with `Billed` = `Remaining` = amount, `Payment status` = `Outstanding`; accordion count/total increase.

### Partial pay
Pay with amount < remaining → confirm `Confirm fee/fine payment` (`… will be partially paid`) → callout `1 fee/fine for {billed amount} has been successfully partially paid for {name}`; account stays `Open`, `Remaining` = old − paid; details row `Action = Paid partially`, `Amount`, `Balance`.

### Refund gating
`Refund` active only when the selection has a paid or transferred amount not yet refunded; after full refund it becomes inactive for that charge; selecting a never-paid charge with others triggers `Deselect to continue` (C15188–C15190, RefundActionValidationService).

### Cover every entry point
Each action from: Open-tab and All-tab page buttons, row ellipsis, Fee/fine details buttons (and Loans/Check in ellipsis for `New fee/fine`). Selection matrix: none → disabled; closed only → disabled; open+closed → `Alert details`; multiple open → summary count + amount split (see rule 7).

### Details page reload
Open Fee/fine details, refresh and paste the URL in a new tab → page loads, URL unchanged (UIU-3428, C808511).

---

## Lost Item Fee Refund Mechanics on Return/Renew [C11031, C11032, C15201]

- Declared lost and Aged to lost are handled the same; check-in or renewal resolves them. If the lost item fee was partly paid/transferred/waived, check-in shows `Check in aged to lost item` (Cancel/Confirm).
- Paid amount → `Credit` then `Refund` (same amount); transferred amount → Credit + Refund back to the transfer account; outstanding remainder → `Cancelled item returned`; waived amount is never reversed. Balance → 0.00, fee closes.
- History strings: `Credited/Refunded fully - Lost item found`, `Cancelled item returned`.
- Lost item processing fee follows the policy toggle "If lost item returned or renewed, remove lost item processing fee" (Yes → same mechanics; No → untouched).
- Fees are not refunded when the item is returned after the policy's "No fees/fines shall be refunded if a lost item is returned more than" limit (C375276, C375277, C375286, C375287).
- Already `Lost and paid` items: separate scenario.

## Aged to Lost — Exclusions and Recall Interval [C15198, C196743]

- Claimed-returned loans are skipped by the aged-to-lost billing job.
- Two aging tracks in the Lost item fee policy: regular ("Items aged to lost after overdue" / "Patron billed after aged to lost") and recall ("Recalled items aged to lost after overdue" / "Patron billed for recall after aged to lost"); leaving a pair blank disables aging for that category only.

## Actual cost [section 20624]

- Actual-cost lost items appear on `Lost items requiring actual cost` (Users > Actions); `Bill actual cost` → `Actual cost to bill {patronName}` → `Confirm actual cost to bill {patronName}`; amount must satisfy `Actual cost to bill patron must be greater than 0.00 and less than 9999.99` (`lostItems.feeFineAmount.error`; C374121, C374123); `Do not bill` closes the loan as `Lost and paid` without a charge (docs feesfines.md; C374125, C375156).
- A record already billed / not-billed by another user cannot be billed again (C374124, C375161). Entries are removed when the item is returned/renewed within the refund limit (C375274–C375285).
- Set cost and Actual cost lost item fees both close on check-in, including after the policy was switched between them (CIRC-2130; C1464246–C1464248, C1504471).

## Reminder Fees [section 25512; CIRC-1527/1528/1599, MODFEE-345/356/358]

Configured in an Overdue fine policy (`Settings > Circulation > Fee/fine > Overdue fine policies`, "Reminder fees" section, `Add reminder fee`): per row Interval + days/weeks, Amount, Notice method, Notice template. Created by an overnight process in sequence; billed when the item becomes overdue (overdue fines are billed on return — docs feesfines.md). Return stops further reminders (C411799); "Allow renewal of items with reminder fee(s)" gates renewal (C451528; else `Renewals not allowed for loans with reminders.`, `errors.renewWithReminders`); renewing resets to a new first reminder (C451530); closed days shift the schedule (C451535, C451537); mandatory-field validation (C411797). Reminder fee details page shows the loan's returned date (UIU-3187). Automated type name `Reminder fee` (MODFEE-358).

---

## Key Business Rules for Test Cases

1. Every manual charge type, payment method and transfer account belongs to a Fee/fine owner; an owner is a precondition for any charge (C442, C445, C5556; docs feesfines.md).
2. Automatic fees pick the owner via item effective location → primary service point → owner; with no owner for that service point the fee is not billed (`Lost item fee(s) not billed to patron`) (docs feesfines.md; en.json `feefines.errors.notBilledMessage`).
3. A `Shared` owner cannot have associated service points; other owners may have none (C350615).
4. An owner with fees/fines attached cannot be deleted; a manual charge type used by any account cannot be deleted (en.json `owners.cannotDeleteOwner`, `feefines.disabledItem`; C440, C442).
5. Manual charge type names must be unique per owner and across the Shared owner, and cannot reuse an automated type name (en.json `feefines.errors.*`).
6. Pay / Waive / Transfer can be full or partial; partial keeps the account `Open` with `Paid/Waived/Transferred partially`, full closes it with `… fully` (mod-feesfines ActionService; C459, C5575).
7. A bulk action amount is split evenly (floor to the cent) across selected accounts in ascending order of remaining; accounts smaller than the even share are fully covered and the rest is re-split; leftover cents go one by one to accounts with room (SplitEvenlyRecursively.java; C459 sc.15).
8. Action amount must be > 0, numeric, and ≤ total remaining (refund: ≤ paid + transferred − already refunded); API returns 422 otherwise (DefaultActionValidationService, RefundActionValidationService; Karate accounts.feature).
9. Pay / Waive / Transfer / Cancel are rejected on a closed account (`Fee/fine is already closed`); Refund is allowed on closed accounts (validation services; Karate "check-refund should be allowed for a closed account").
10. Refund is available only for paid or transferred amounts; a never-paid charge cannot be refunded, and a fully refunded charge cannot be refunded again (C15188, C15190; RefundActionValidationService).
11. A refund refunds paid amounts first, then transferred amounts per transfer account, creating a `Credited …` action followed by a `Refunded …` action for each, and sets the account status back to `Open` (RefundActionService.java; Karate manualFeeFines.feature).
12. Every Pay / Waive / Transfer / Refund needs a second confirmation `Confirm fee/fine {payment|waive|transfer|refund}` before it is posted (FeeFineActions.js; C356793, C458).
13. Cancel as error (`Error`) is available only while nothing has been paid / waived / transferred; it becomes available again after a full payment is fully refunded; a comment is mandatory (C17157, C470, C356793; `accounts.cancellation.error.comment`).
14. Selecting closed (or, for refund, unpaid) fees/fines together with valid ones opens `Alert details` and requires `Deselect to continue` (en.json `accounts.warning.*`; C15190, C17134).
15. Owner "Comment required" settings make `Additional information for staff` mandatory for that action (C448, C5577, C15197; `accounts.error.comment`).
16. `Notify patron` and `Additional information for patron` appear only when a default or type-specific action/charge notice is configured; they are absent otherwise (C6643, C6645, C15192; Cypress `verify-refund-behavior-*`).
17. A staff user without a current service point cannot create or view patron fees/fines (`Access denied` modal) (en.json `errorModal.currentUserServicePointAbsentee.*`; Cypress `check-user-cannot-create-fee-fine-without-service-point.cy.js`).
18. A manual fee/fine with an unknown item barcode cannot be saved: `No item with barcode {barcode} could be found`, charge buttons disabled (UIU-2701).
19. Amount inputs in action modals are number fields — whitespace and letters cannot be entered; comma decimals are not supported (UIU-2836, UIU-3206).
20. `Fees/Fines: Can pay` / `Fees/Fines: Can waive` allow only that action; refund, transfer and cancel need `Users: Can create, edit and remove fees/fines` (`Data - UI-Users Feesfines Actions - Manage`); action buttons are hidden without permission (UIU-3109, UIU-3156; package.json; C494013, C494014).
21. Creating a fee/fine from an open loan works with `Data - UI-Users Feesfines Actions - Manage` alone (inventory item lookup is included) (UIU-3602).
22. Lost-item fees on a Claimed-returned loan are `Suspended claim returned` and blocked from actions; fees of an earlier, closed loan of the same item stay actionable (UIU-3125; C491280).
23. Resolving a claim as Missing should cancel the suspended lost item fee and processing fee (C196748); this is done by a second UI call and has failed intermittently (UIU-3490/MODFEE-411 Won't Do, UIU-3573 Duplicate; backend move planned in MODFEE-444/UIU-3544).
24. Resolving a lost item by check-in or renewal reverses paid/transferred amounts as Credit + Refund, cancels the outstanding rest as `Cancelled item returned`, never reverses waived amounts; the processing fee follows its own policy toggle (C11031, C11032, C15201, C15209 for a former declared-lost item already `Lost and paid`).
25. Claimed-returned loans are excluded from aged-to-lost billing; recalled items age to lost on their own interval (C15198, C196743).
26. When all fees of a lost loan are resolved by pay / waive / cancel, the loan closes and the item becomes `Lost and paid` (docs feesfines.md; C366535, C410831).
27. When a fee/fine is resolved (including by transfer to a bursar), the loan closes and the history list may show a `Returned date` equal to the resolution date — expected behaviour (MODFEE-433, Won't Do).
28. Reminder fees are configured in the Overdue fine policy, created overnight in sequence, stop on return, reset on renewal, shift on closed days, and can block renewal (Reminder Fees section; C411799, C451528, C451530, C451535).
29. Actual cost to bill must be > 0.00 and < 9999.99; `Do not bill` closes the loan as `Lost and paid` with no charge (C374121, C375156; en.json `lostItems.feeFineAmount.error`).
30. Fee/fine details page must load on refresh and direct URL (UIU-3428, C808511); the Refund action must be enabled right after a fresh login when a paid fee/fine is selected on the history page (UIU-3549, C15189).
31. Financial-transactions and cash-drawer reports require a Start date (and owner / service point respectively) and End date ≥ Start date; the Financial report is for one owner and 0..n service points (C343207, C343311, C343314, C343324).
32. Deleting an item does not delete its fee/fine; the account keeps stored item data but the UI may show an error for it (docs feesfines.md) — avoid deleting items with fees in test data.
33. On Check out, the borrower's `Fees/fines owed` amount is a link to the patron's fees/fines page only with `Check out: View fees/fines` + `Users: Can view fees/fines and loans`; with only `Check out: Check out circulating items` it is plain text (C388524, C388525; UICHKOUT-802).
34. Overdue fine amount = interval count × rate from due date to returned/renewed date, minus the Loan-policy grace period, capped by `Maximum overdue fine` (5 min × 5.00 = 25.00 → billed 15.00 at max 15.00); recalled items use `Overdue recall fine` / `Maximum recall overdue fine`, and `Ignore grace periods for recalls` = No keeps the grace period (C9270, C9273; CIRC-524, CIRC-548, UICHKIN-125).
35. `Forgive overdue fine if item renewed` = Yes → no overdue fine on renewal; = No → fine billed for the overdue minutes (C9271; CIRC-677).
36. Overdue-fine record: type `Overdue fine`, owner via item effective location → primary service point → owner, `Payment status` = `Outstanding`, history action `Overdue fine`, `Created at` = check-in service point, `Source` = staff user who checked in, `Transaction information` blank (C9270, C9273).
37. A Declared lost or Aged to lost item that is returned or renewed still gets an overdue fine when the Lost item fee policy says `If lost item returned or renewed = Charge overdues based on returned date up to maximum` (1.00 per minute from due date to returned date in the case) (C15207 body; C15208, C15203, C15204 by title only).
38. Actual cost on declared lost: the `Lost item processing fee` is billed immediately and the loan's `Fees/Fines Incurred` equals it (25.00 in the case); the actual-cost amount itself is billed later from `Lost items requiring actual cost` (C365133; CIRC-1623, CIRC-2522, CIRC-2523, CIRC-2530).
39. Aged to lost with Actual cost: the item is aged to lost only if the policy has a `Lost item processing fee` > 0; with no processing fee it is excluded and stays checked out (C235864 / C235865; TestRail-only, outdated risk — see Known Gaps).
40. After an actual-cost fee is billed and paid, the item shows `Lost and paid` (no borrower, loan date or due date) and the closed loan's action table has `Action` = `Closed loan`, `Item status` = `Lost and paid`, `Source` = staff who closed the fee/fine (C359236; UIU-1605, CIRC-1556, CIRC-1566, CIRC-2044, CIRC-2066).

---

## Edge cases from recent bugs (Jira, resolved ≥ 2024-06)

- UIU-3549 (Done): Refund disabled after fresh login for fully paid/transferred fee selected on history page — regression check for C15189/C15190.
- UIU-3428 (Done): details page crash on reload/URL.
- UIU-3602 (Done): 403 on `/inventory/items` when creating a fee from an open loan with only `Feesfines Actions - manage`.
- UIU-3193 (Done): `/` in item barcode broke `New fee/fine` from Check in.
- UIU-3125 (Done): previous-loan fees blocked after the item is claimed returned on a new loan.
- UIU-3094 (Done): `Total paid amount` showed `$NaN` in Refund modal.
- UIU-3398 (Won't Do, Okapi only): bulk transfer disabled because `Users: Can create, edit and remove fees/fines` lacks `feesfines.accounts-bulk.check-transfer.post` (still absent in package.json).
- MODFEE-416 (Won't Do): hourly overdue fines with closed hours — calculation per docs (minutes rounded up to interval).
- MODFEE-388 / MODFEE-450 (Cannot Reproduce): duplicate aged-to-lost bills.
- CIRC-2130: Set/Actual cost fees close on check-in; MODAUD-324: fee/fine actions missing from Circulation log (fixed).

---

## Authoring style (measured 2026-07-23)

Fees&Fines has the shortest cases in the project — median ~2 steps, `Type` Other ~79%, Functional ~20%. Put heavy setup (owner, policies, circulation rule, overdue item) in Preconditions, then 1–3 tight steps with one outcome. Recent cases list both legacy permissions and Eureka capability sets (`For Eureka environments, use following capabilities/sets (type - resource - action)` — C375239, C808511). Reminder-fee and lost-item lifecycle cases run longer.

---

## TestRail — cases created/updated since 2026-07-21

- C1504471, C1464248, C1464247, C1464246 — Set/Actual cost lost item fee closed on check-in (CIRC-2130).
- C1395066, C1405034 — Set cost lost item fee / processing fee suspended and refunded when claimed returned (UIU-3603, CIRC-2637 in refs).
- C375239 (UIU-1097), C808511 (UIU-3428) — updated with Eureka capability sets.
- Outdated: C3646 `Fees/Fines: Can pay` (section 183) has only a precondition note "Will be renamed to User can pay fee/fines"; C3647 `Fees/Fines: Can waive` is marked OBSOLETE — do not copy as models (Waive-only permission behaviour is in C494014).
- Bursar section 11060 (C196783, C196787, C400631, C400632, C400641) is site-specific (Cornell / Alabama / Duke bursar files) and C400625 has only a one-line precondition (5-column CSV with separators and EOL, no steps); bodies of the others were not read.

---

## Known Gaps / Items to Verify

- [ ] Capability set names for Waive-only (`UI-Users Manual-Waive`), Charge (`UI-Users Manual-Charge`), reports and fee/fine settings are NOT confirmed (marked `(capability set name not confirmed — verify)`; not in the Cypress dictionary or any case read). Confirmed: `UI-Users Feesfines - View`, `UI-Users Feesfines Actions - Manage` (C375239, C491280, C808511 only — the Cypress dictionary instead has `UI-Users Fee-Fine-Actions - Manage`, so verify which set exists on the env), `UI-Users Manual-Pay - Execute` (dictionary + C375239), `UI-Users Lost-Items-Requiring-Actual-Cost - Execute` (C359236). Fee/fine actions history = `UI-Users Fee-Fine-Actions - Manage` is mapped to its legacy permission by naming only.
- [ ] `Users: Can create, edit and remove fees/fines` lacks `feesfines.accounts-bulk.check-transfer.post` / `check-refund.post` (package.json; UIU-3398 closed as Okapi-only) — verify bulk Transfer/Refund with only this capability set on Eureka.
- [ ] Success callout shows the account's billed `amount`, not the action amount, and uses `fee/fine` + count 1 per account in bulk (FeeFineActions.js) — confirm in env before asserting the amount in a callout.
- [ ] Claimed returned → Mark as missing fee cancellation is intermittent until MODFEE-444 / UIU-3544 (Draft) ship.
- [ ] UIU-3603 / CIRC-2637 (refs of C1395066) not read — check scope before editing claimed-returned cases.
- [ ] `{paymentName}` value in the Charge only callout (ChargeFeeFine.js passes form data; expected `outstanding`) — verify in env.
- [ ] en_US.json was not compared with en.json in this refresh; all strings above are from en.json (master, 2026-11-16).
- [ ] Release mapping (ui-users 13.0.x = Trillium, 13.1.0 = Umbrellaleaf) inferred from CHANGELOG dates; Jira fixVersions hold module versions only.
- [ ] `Partially waive` / `Partially transfer` summary verbs (en.json) differ in form from `Partially paying` / `Partially refunding` — real strings, assert as written.
- [ ] Aged-to-lost job timing conflicts between cases: C235864 says the job runs hourly on the hour with billing 30 min later; C235865 says every 30 minutes — verify in the environment before asserting timing. C235864 / C235865 (actual cost needs a processing fee to age to lost) predate the actual-cost processing work (CIRC-2130) — verify the rule on current master (rule 39).
- [ ] TestRail second pass read bodies of C3646, C388524, C388525, C359236, C359591, C365133, C15207, C15209, C400625, C9270, C9271, C9273, C235864, C235865, C11149, C375239, C491280, C808511. NOT read (titles only): sections 90 Manual, 91 Pay, 92 Display (except C11149), 1533 Waive, 1141 other cases (C9267-C9269, C9272, C9274, C9275), 25512 Reminder Fees, 1312/1531 other cases, 1535 Settings Circulation, 1536 Manual Patron Blocks, 94 Automated Patron Blocks, 1142 Item Blocks, 17145 Fees/fines patron notices, 16509/16510 report cases beyond those cited, 11060 Bursar (except C400625).
