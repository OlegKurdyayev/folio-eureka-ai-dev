# FOLIO Loans — Business Logic Context

> Built: 2026-11-24 · Release: Umbrellaleaf R2 2026 (master: ui-users 13.1.0 (in progress), mod-circulation 24.6.0-SNAPSHOT) · Builder: build-app-context (refresh) · Previous build: 2026-07-21 (browser-based), delta-refreshed 2026-11-07 — the delta is folded into the sections below.

> Sources (2026-11-24):
> - TestRail project 14 / suite 21: section 97 "Loans" (21 cases) and subsections Additional Loan Comments 25717 (2), Anonymization 1132 (14), Change due date 1287 (4), Declare lost 1290 (4), Claim returned 1294 (5), Renewals 1540 (24), **Automatic renewals 17138 (9, new)**. "Permissions --> Loans" (174) is empty.
> - GitHub: `folio-org/ui-users` (translations/ui-users/en.json, package.json, CHANGELOG.md, src/), `folio-org/stripes-smart-components` (translations — Change due date dialog `cddd.*`), `folio-org/ui-circulation` (translations — loan policy / anonymization settings), `folio-org/mod-circulation` (descriptors/ModuleDescriptor-template.json, ramls/, ErrorCode.java, NEWS.md), `folio-org/folio-integration-tests` (mod-circulation Karate), `folio-org/stripes-testing` (Cypress), `folio-org/docs` (content/en/docs/Users/_index.md).
> - Jira folio-org: UIU + CIRC Story/Bug/New Feature resolved since 2026-06-01 → 476 issues, 115 loan-related, ~40 read in detail (plus MODPATBLK-236/237, UICIRC-1361, UIU-3595, UIU-3544).
> - Confluence (anonymous REST): "User-specific loan anonymization" (page 2129002498, DRAFT, updated 2026-10-26).
> - Tags: `[en.json key]` = exact UI string from translations (source of truth). `[C…]` = asserted in that TestRail case. Where the two differ, en.json wins and the difference is listed in Known Gaps.

---

## What is Loans

A **loan** is created when an item is checked out to a patron and closed when the item is checked in (or resolved via lost/missing/anonymization flows). Staff manage loans from the patron record in the Users app: **Loans** accordion → **Open loans** / **Closed loans** list → per-loan **Loan details** page. Staff actions on an open loan: **Renew**, **Change due date**, **Declare lost**, **Claim returned** (→ **Resolve claim**: Declare lost / Mark as missing), **New patron info**, **New staff info**, **Print due date receipt**, **New fee/fine**. Check out / check in themselves live in the Check out / Check in apps (see check-out.md, check-in.md); loan policies and anonymization settings live in Settings > Circulation (see circulation-settings.md). Since Umbrellaleaf the backend also performs **automatic renewals** (scheduled, loan-policy driven).

---

## Key Terms

| Term | Definition |
|---|---|
| Open loan | Active loan; `status.name` = `Open`. Item is out to the patron. (mod-circulation `ramls/loan.json`) |
| Closed loan | Returned/resolved loan; `status.name` = `Closed`. (mod-circulation `ramls/loan.json`) |
| Loan details | Per-loan page with loan fields + the **Loan action history** table (`loans.history`). |
| Loan action | Row in the loan history (values in "Action values" below). Stored in loan `action` + `actionComment` (mod-circulation `ramls/loan.json`). |
| Declared lost | Staff-marked lost status; bills per Lost item fee policy. (C9191, C10960; CIRC-2130) |
| Aged to lost | System-set lost status (lost item fee policy "Items aged to lost after overdue"). (CIRC-2629; Karate `loans.feature` aged-to-lost scenario) |
| Claimed returned | Patron claims return; loan stays open, lost-item fees are suspended until resolved. (Karate `loans.feature`; docs Users "Mark an item claim returned"; C10960) |
| Override | Renew despite a renewal block (needs renew-through-override capability) or despite a patron block. (C568, C170386) |
| Automatic renewal | Scheduled backend renewal of eligible loans, enabled per loan policy (CIRC-2687, CIRC-2584). |
| Use at location | Reading-room loan (In use / Held); shown in Open loans and Loans accordion (UIU-3405, UIU-3406). |

---

## Loan details / Loans list — structure

- **Loan details buttons**: `Renew`, `Claim returned` (replaced by `Resolve claim` once claimed returned), `Change due date`, `Declare lost`, `Print due date receipt` [en.json `renew`, `loans.claimReturned`, `loans.resolveClaim`, `cddd.changeDueDate` (stripes-smart-components), `loans.declareLost`, `loans.details.printDueDateReceipt`]; plus `New patron info`, `New staff info`, `New fee/fine` [`loans.newPatronInfo`, `loans.newStaffInfo`, `loans.newFeeFine`].
- **Loan details fields** (en.json `loans.details.*`): `Borrower`, `Proxy borrower`, `Loan policy`, `Overdue fine policy`, `Lost item fee policy`, `Renewal count`, `Fees/fines incurred`, `Claimed returned`, `Lost`, `Request queue`, `Location`, `Effective call number string`, `Check in service point`; plus `Item status`, `Due date`, `Loan date`, `Return date`.
- **Action history columns**: `Action Date`, `Action`, `Due date`, `Item status`, `Source`, `Comments` [`loans.columns.actionDate`, `.action`, `.dueDate`, `.itemStatus`, `.source`, `.comments`]. `Source` = staff "Lastname, firstname middlename" linked to the user record, or `System` [`loans.action.source.system`] for system actions (aged to lost, automatic renewal — C1621350).
- **Open/Closed loans list**: toggle `Open` / `Closed` [`loans.open`, `loans.closed`]; columns include `Item title`, `Barcode`, `Due date`, `Loan date`, `Return date` (closed), `Renewal count`, `Fees/fines incurred`, `Contributors`, `Loan policy`, `Use at location` (open loans only, only with use-at-location view permission — UIU-3405); row checkbox `Select loan`, header `Select all loans`. Bulk buttons: `Renew`, `Claim returned`, `Change due date`, `Print due date receipt`, export CSV; per-row ellipsis menu mirrors single actions (C562).
- **Loans accordion** on the user record: `{n} open loans` / `{n} closed loans` links, claimed-returned count in parentheses `({n} claimed returned)` [`loans.numClaimedReturnedLoans`], use-at-location split `Held: {n}` / `In use: {n}` [`loans.numOpenLoans.held`, `.inUse`] (UIU-3406, counts are not links — UIU-3475).
- **Loan with deleted item** (UIU-2531, C656307): item fields show `-`; Renew, Claim returned, Change due date, Print due date receipt, Declare lost, Mark as missing are disabled; the row checkbox in Open loans is disabled; ellipsis items Item details / Renew / Claim returned / Change due date / Print due date receipt / Declare lost / New fee/fine / Mark as missing are disabled.

---

## Status values

### Loan status (`status.name`)
`Open`, `Closed` (mod-circulation `ramls/loan.json`).

### Item status values seen on loans (en.json `item.status.*` and loan history)
`Checked out`, `Declared lost`, `Aged to lost`, `Claimed returned`, `Lost and paid`, `Missing`, `Long missing`.

### Action values (loan history) [en.json `data.loanActionMap.*`]
`Checked out`, `Checked out through override`, `Checked in`, `Checked in (returned by patron)`, `Checked in (found by library)`, `Renewed`, `Renewed through override`, `Due date changed`, `Hold requested`, `Recall requested`, `Declared lost`, `Aged to lost`, `Claimed returned`, `Marked as missing`, `Closed loan`, `Patron info only`, `Patron info (SUPERSEDED)`, `Staff info only`, `Staff info (SUPERSEDED)`, `Reminder fee`, `Held for use at location`, `Picked up for use at location`, `Unknown action`.

### Transitions

```
Checked out
  ├─ Renew (manual or automatic) → Checked out, new due date per loan policy; renewal count +1
  ├─ Change due date → Checked out, chosen date/time ("Due date changed")
  ├─ Declare lost (comment required) → Declared lost (bills per lost item fee policy)
  ├─ Claim returned (comment required) → Claimed returned (lost fees suspended)
  ├─ (system) Aged to lost → Aged to lost
  └─ Check in → loan Closed
Declared lost / Aged to lost
  ├─ Renew → blocked; Override & renew → Checked out, due date auto-calculated (C9181, C9192)
  ├─ Change due date → blocked (row alert)
  ├─ Claim returned → Claimed returned
  └─ $0 lost fee + "close the loan after" → Lost and paid, loan Closed (C10948)
Claimed returned
  ├─ Renew / Change due date → blocked, NO override (C9181, C10958)
  ├─ Resolve claim → Declare lost → Declared lost (loan stays open)
  ├─ Resolve claim → Mark as missing → Missing, loan Closed
  └─ Check in (Found by library / Returned by patron) → loan Closed
```

---

## API surface (mod-circulation `descriptors/ModuleDescriptor-template.json`, master 24.6.0-SNAPSHOT)

| Method + path | Interface | permissionsRequired | Request body (ramls/) |
|---|---|---|---|
| GET `/circulation/loans`, GET `/circulation/loans/{id}` | circulation 14.10 | `circulation.loans.collection.get` / `circulation.loans.item.get` | — |
| PUT `/circulation/loans/{id}` | circulation 14.10 | `circulation.loans.item.put` | loan |
| POST `/circulation/renew-by-barcode` | circulation 14.10 | `circulation.renew-by-barcode.post` | `itemBarcode`*, `userBarcode`*, `servicePointId`, `overrideBlocks` |
| POST `/circulation/renew-by-id` | circulation 14.10 | `circulation.renew-by-id.post` | — |
| POST `/circulation/bulk-renew-open-loans` | circulation 14.10 | `circulation.bulk-renew-open-loans.post` | (automatic renewal backend) |
| POST `/circulation/loans/{id}/change-due-date` | change-due-date 0.1 | `circulation.loans.change-due-date.post` | `dueDate`* |
| POST `/circulation/loans/{id}/declare-item-lost` | declare-item-lost 0.3 | `circulation.loans.declare-item-lost.post` | `declaredLostDateTime`*, `servicePointId`*, `comment` |
| POST `/circulation/loans/{id}/claim-item-returned` | claim-item-returned 0.2 | `circulation.loans.claim-item-returned.post` | `itemClaimedReturnedDateTime`*, `comment` |
| POST `/circulation/loans/{id}/declare-claimed-returned-item-as-missing` | claim-item-returned 0.2 | `circulation.loans.declare-claimed-returned-item-as-missing.post` | `comment`* |
| POST `/circulation/loans/{id}/add-info` | add-info 0.1 | `circulation.loans.add-info.post` | `action`* (`staffInfoAdded` / `patronInfoAdded`), `actionComment`* |
| POST `/loan-anonymization/by-user/{userId}` | loan-anonymization 0.1 | `circulation.loans.collection.anonymize.user.post` | — |
| POST `/circulation/scheduled-anonymize-processing`, `/circulation/scheduled-loan-anonymization-preference-sync` | _timer | — | scheduled jobs |

- Override renewal: `circulation.override-renewal-block.post` (sub-permission of the UI override set).
- UI uses `circulation-bff` for loan lists/details (`GET /circulation-bff/loans`, UIU-3392 — breaking in ui-users 13.0.0) and for declare-item-lost / claim-item-returned / mark-as-missing (UIU-3414, UIU-3413, UIU-3444) — DevTools assertions should expect `/circulation-bff/loans/{id}/declare-item-lost` etc. (CIRC-2673 stack trace).
- Renewal 422 errors carry a machine-readable `code` (CIRC-2632) mapped to UI strings (ui-users `src/constants.js` `ERROR_MESSAGE_TRANSLATION_ID_BY_BACKEND_ERROR_CODES`). Renewal-related codes (mod-circulation `ErrorCode.java`): `ITEM_NOT_LOANABLE`, `ITEM_AGED_TO_LOST_NOT_RENEWABLE`, `ITEM_DECLARED_LOST_NOT_RENEWABLE`, `ITEM_CLAIMED_RETURNED_NOT_RENEWABLE`, `LOAN_RENEWAL_LIMIT_REACHED`, `LOAN_NOT_RENEWABLE`, `LOAN_POLICY_PROFILE_NOT_RECOGNIZED`, `LOAN_POLICY_RENEW_FROM_NOT_RECOGNIZED`, `LOAN_POLICY_RENEWAL_PERIOD_NOT_RECOGNIZED`, `LOAN_POLICY_RENEWAL_PERIOD_INTERVAL_NOT_RECOGNIZED`, `LOAN_POLICY_RENEWAL_PERIOD_DURATION_INVALID`, `RENEWAL_REQUEST_ITEM_BARCODE_REQUIRED`, `RENEWAL_REQUEST_USER_BARCODE_REQUIRED`, `RENEWAL_BLOCKED_BY_RECALL`, `RENEWAL_BLOCKED_BY_REMINDER_FEES`, `RENEWAL_BLOCKED_BY_HOLD_REQUEST`, `RENEWAL_DATE_OUTSIDE_LOAN_POLICY_DATE_RANGES`, `RENEWAL_DATE_OUTSIDE_FIXED_LOAN_POLICY_DATE_RANGES`, `RENEWAL_WOULD_NOT_CHANGE_DUE_DATE`, `FIXED_LOAN_POLICY_HAS_ALTERNATE_RENEWAL_PERIOD_FOR_HOLDS`, `FIXED_LOAN_POLICY_HAS_RENEWAL_PERIOD`, `CALENDAR_OPEN_PERIODS_NOT_FOUND`, `CALENDAR_TIMETABLE_ABSENT`, `OVERRIDE_RENEWAL_COMMENT_REQUIRED`, `OVERRIDE_RENEWAL_DUE_DATE_REQUIRED`, `OVERRIDE_RENEWAL_DUE_DATE_REQUIRED_WHEN_DUE_DATE_UNCHANGED`, `OVERRIDE_RENEWAL_NOT_ALLOWED_FOR_CURRENT_LOAN`, `INSUFFICIENT_OVERRIDE_PERMISSIONS`. Overridable blocks are reported as `overridableBlock.name` = `renewalBlock` / `renewalDueDateRequiredBlock` with `missingPermissions` (UIU-977, CIRC-2632, CIRC-2253).

---

## Exact UI Texts

> en.json keys are `ui-users.*` unless prefixed `ssc:` (stripes-smart-components `cddd.*`) or `ui-circulation:`. `<b>`/`<strong>` render bold; `{x}` are placeholders.

### Renew
- Single renew success callout: `The loan for {title} was successfully renewed.` [`loans.item.renewed.callout`; sent from `withRenew.js` showSingleRenewCallout].
- Multi-item renew → `Renew Confirmation` modal [`brd.renewConfirmation`] (fixed: UIU-3625, C567), summary `{count} item(s) successfully renewed` / `{count} item(s) not renewed.` [`brd.itemSuccessfullyRenewed`, `brd.itemNotRenewed`], column `Renewal status` [`brd.header.renewalStatus`] with `Item successfully renewed` [`brd.successfulRenewal`] or `Item not renewed:` [`brd.failedRenewal`] followed by the reason on the next line; `Close`, and `Override` only for users with the override capability (C568).
- Failure reasons (en.json `messages.*`, chosen by backend error code — UIU-3537, CIRC-2632):

| Code | UI text [key] |
|---|---|
| LOAN_NOT_RENEWABLE | `Loan is not renewable.` [`messages.loanNotRenewable`] (C568) |
| LOAN_RENEWAL_LIMIT_REACHED | `Loan at maximum renewal number.` [`messages.loanRenewalLimitReached`] (C569) |
| RENEWAL_WOULD_NOT_CHANGE_DUE_DATE | `Renewal would not change the due date.` [`messages.renewalWouldNotChangeDueDate`] (C571) |
| RENEWAL_BLOCKED_BY_HOLD_REQUEST | `Items with this loan policy cannot be renewed when there is an active, pending hold request.` [`messages.renewalBlockedByHoldRequest`] (C6667) |
| RENEWAL_BLOCKED_BY_RECALL | `Items cannot be renewed when there is an active recall request.` [`messages.renewalBlockedByRecall`] (C9180) |
| RENEWAL_BLOCKED_BY_REMINDER_FEES | `Renewals not allowed for loans with reminders.` [`messages.renewalBlockedByReminderFees`] |
| ITEM_NOT_LOANABLE | `Item is not loanable.` [`messages.itemNotLoanable`] (C15492) |
| ITEM_AGED_TO_LOST / DECLARED_LOST / CLAIMED_RETURNED_NOT_RENEWABLE | `Item is Aged to lost.` / `Item is Declared lost.` / `Item is Claimed returned.` [`messages.item…NotRenewable`] (C9181, C9192, C10958) |
| RENEWAL_DATE_OUTSIDE_(FIXED_)LOAN_POLICY_DATE_RANGES | `Renewal date falls outside of date ranges in the loan policy.` / `… in fixed loan policy.` (C570, C1453709) |
| CALENDAR_TIMETABLE_ABSENT / CALENDAR_OPEN_PERIODS_NOT_FOUND | `Calendar timetable is absent for requested date.` / `Calendar open periods are not found.` (C1538662) |
| FIXED_LOAN_POLICY_HAS_RENEWAL_PERIOD / …ALTERNATE_RENEWAL_PERIOD_FOR_HOLDS | `Item's loan policy has fixed profile but renewal period is specified.` / `Item's loan policy has fixed profile but alternative renewal period for holds is specified.` |
| LOAN_POLICY_* not recognised | `Profile {profile} in the loan policy is not recognised.`, `Cannot determine when to renew from.`, `The loan period in the loan policy is not recognised.`, `The interval {interval} in the loan policy is not recognised.`, `The duration {duration} in the loan policy is invalid.` |
| RENEWAL_REQUEST_*_BARCODE_REQUIRED (API only — C1505018) | `Renewal request must have an item barcode.` / `Renewal request must have a user barcode.` |
| OVERRIDE_* (API only — C1528148) | `Override renewal request must have a comment.`, `New due date must be specified when due date calculation fails.`, `New due date is required when renewal would not change the due date.`, `Override renewal does not match any of expected cases: ...`, `Insufficient override permissions.` |

- Other renewal error wrappers: `Loan cannot be renewed because: {message}.` [`errors.loanNotRenewedReason`], `{message} Please review {policyName} before retrying renewal.` [`errors.reviewBeforeRenewal`], `Loan not renewed` [`loanNotRenewed`].

### Override & renew [C568, C9181, C9192]
- Modal `Override & renew` [`brd.overrideAndRenew`]: date/time pickers (required unless auto-calculable), table with `Renewal status`, `New due date` [`brd.header.newDueDate`], title, item status, due date, requests, barcode, effective call number string, renewal count, loan policy; `New due date` cell = `Select due date above` [`override.emptyNewDueDate`] or `Due date will be calculated automatically` [`override.autoNewDueDate`] (lost items); required additional information (placeholder `Enter more information about the override (required)` [`additionalInfo.placeholder`]); `Cancel` / `Override` (inactive until a loan is checked and required fields are filled).
- Result: history row `Renewed through override`, Source = overriding staff, Comments = justification; renewal count +1 (C568).

### Override patron block on renewal [C170386]
- Modal `Patron blocked from renewing` with `Reason for block:`; buttons `Override`, `Close`, `View block details`; max three blocks shown. `Override` opens `Override patron block` [`blocks.modal.overridePatronBlock.header`] with required `Comment*`; `Save & close` renews and stores the comment on the loan.

### Change due date (stripes-smart-components dialog) [ssc `cddd.*`; C565, C566, C572, C581, C9192, C10958]
- Header `Change due date`, `{n} items selected.`, `Date`, `Time`; columns `Alert details`, `Title`, `Item status`, `Current due date`, `Requests`, `Barcode`, `Effective call number string`, `Loan policy`; buttons `Save and close`, `Close`.
- Pre-save warnings: `Item is Declared lost`, `Item is Claimed returned`, `Item is Aged to lost`, `Item has been recalled`, `Item has a request queue`; summary `{n} items have warnings.`; date warnings `New due date is after patron's expiration.`, `New due date is when the library is closed.`, `New due date is in the past.`
- Save results: `Due date & time has successfully been changed` / `Due date & time has been successfully changed for {n} items.` / `{failed} due dates not changed. {succeeded} due dates successfully changed.`; per-row failures `Failed to change due date: item is Declared lost`, `Due date change failed: item is Claimed returned`, `Due date change failed: item is Aged to lost`, generic `Failed to change due date`.

### Declare lost [en.json; C9191, docs]
- Modal title `Confirm item status: Declared lost` [`loans.confirmDeclaredLost`]; body `{title} ({materialType}) (Barcode: {barcode}) will be Declared lost.` + open-request sentence (`There may be open requests for this item.` / `There is {n} open request for this item.`) [`loans.declareLostDialogBody`]; placeholder `Enter more information about the lost item (required)`; `Cancel` / `Confirm`.
- After confirm: `Item status` = `Declared lost`; `Lost` field = date/time; history row `Declared lost`, due date unchanged, Comments = entered text; `Declare lost` button inactive (C9191). Single loan only — no bulk declare lost (docs).

### Claim returned [en.json; C10959, C10960]
- Single modal title `Confirm item status: Claimed returned` [`loans.confirmClaimedReturned`]; body `{title} ({materialType}) (Barcode: {barcode}) will be Claimed returned.` + open-request sentence [`loans.claimReturnedDialogBody`]; required additional information; `Cancel` (active) / `Confirm` (inactive until text entered).
- Bulk: `Confirm claim returned` [`bulkClaimReturned.preConfirm`], `{n} item(s) will be claimed returned.`, placeholder `Enter more information about the claim (required)`; result modal `Claim returned confirmation` with `Claim returned status` = `Item successfully claimed returned` / `Item could not be claimed returned: item is already claimed returned`; summary `{n} item(s) successfully claimed returned.` / `{n} item(s) not claimed returned.` [`bulkClaimReturned.*`].
- After confirm: `Item status` = `Claimed returned`; `Claim returned` → `Resolve claim` dropdown; `Renew` and `Change due date` inactive; accordion shows `({n} claimed returned)`.

### Resolve claim [en.json; C10960, docs]
- Options `Declare lost` / `Mark as missing` (also in the loans-list ellipsis menu).
- Declare lost: same `Confirm item status: Declared lost` modal; result Declared lost, `Fees/fines incurred` per lost item fee policy, `Claimed returned` field → `-`, patron note `Claimed returned item marked declared lost` (C10960).
- Mark as missing: title `Confirm item status: Missing` [`loans.confirmMissing`]; body `{title} ({materialType}) (Barcode: {barcode}) will be marked as Missing, and the patron will not be billed.` [`loans.markAsMissingDialogBody`]; result item `Missing`, loan **closed**, all buttons inactive, `Claimed returned` → `-`, patron note `Claimed returned item marked missing` (C10960).

### Manual anonymization (Closed loans) [en.json; C9217, C422021]
- Button `Anonymize all loans` [`anonymize`]; modal `Anonymize all loans?` / `All loans for this user will be anonymized. The loans will no longer appear in the user’s closed loans. Anonymizing loans cannot be reversed.` / `Cancel` / `Confirm` [`anonymization.confirmation.*`].
- Partial failure: `Anonymization prevented` + `{amount} loan(s) had associated fees/fines and could not be anonymized.` [`anonymization.error.*`].
- Fee/fine history link on anonymized loans: `Loan details (anonymized)` [`accounts.history.button.loanAnonymized`]; label `Anonymized` [`details.label.loanAnonymized`].

### User-specific anonymization preference (new, Umbrellaleaf — UIU-3595 In QA, UICIRC-1361 In QA)
- User record > Extended information: radio set `Anonymize closed loans` = `According to library policy` (default) / `As early as possible` / `Never` [`extended.loanAnonymizationPreference.*`]; shown only when the Circulation setting allows user-specific anonymization.
- Settings > Circulation > Loan anonymization: `User-specific loan anonymization`, `Do not allow user-specific loan anonymization`, `If a user requests anonymization as early as possible, then anonymize closed loans` [ui-circulation `settings.loanHistory.userSpecific*`].

### Additional loan comments [en.json; C397378, C397379, docs]
- Buttons `New patron info` / `New staff info`; modals `Add new patron information` / `Add new staff information`; bodies `Add new patron information for {title} ({materialType}) (Barcode: {barcode}). This will supersede any patron information added previously.` / `Add new staff information for {title} ({materialType}) (Barcode: {barcode}).`; `Save & close`.
- History action labels `Patron info only` / `Staff info only`; previous entries relabelled `Patron info (SUPERSEDED)` / `Staff info (SUPERSEDED)` (ui-users `src/views/LoanDetails/LoanDetails.js` appends `.superseded`).

### Automatic renewals — loan policy fields [ui-circulation en.json]
`Automatic renewals` (info popover `Automatic renewals are available for loans with intervals of one day or longer.`), `Attempt first renewal` → `On due date` / `day(s) before due date`, `Attempt additional retries for`.

---

## Settings that drive loan behaviour (owned by circulation-settings.md)

- Loan policy: Loanable, Loan profile (Rolling / Fixed + Fixed due date schedule), Loan period, Closed library due date management, Renewable, Unlimited renewals, Number of renewals allowed, Renew from (system date / current due date), Renewal period different from original loan, Alternate loan period at renewal for items with active pending hold request, Recall return interval, Minimum guaranteed loan period for recalled items, **Automatic renewals** block (C1621350, C1621420, C579, C6667, C466157–C466159).
- Lost item fee policy: aged-to-lost interval, set cost / actual cost, processing fee, "close the loan after" (C10948, C10949, CIRC-2130).
- Loan anonymization: Never / Immediately after loan closes / Retain for a set duration; fee/fine branch; opt-in user-specific section (C9218–C9235, UICIRC-1361).
- Scheduled loan anonymization runs hourly and processes up to 50k loans per run (`SCHEDULED_ANONYMIZATION_NUMBER_OF_LOANS_TO_CHECK`) — CIRC-2459 comment.

---

## Capability Sets (Eureka)

> Legacy permission (ui-users package.json) → capability set. Confirmed = the exact name appears in the Cypress dictionary (`cypress/support/dictionary/capabilitySets.js`) or in a TestRail case precondition. Every other name below is a candidate built from the permission name by the Eureka naming convention (`<module>.<resource>.<action>` → `<Type> - UI-Users <Resource> - <Action>`) and is NOT confirmed — do not put it in case preconditions as fact; verify in the environment first. Permission internal names / display names are confirmed by Cypress `permissions.js`.

| Action | Legacy permission (displayName) | Capability set |
|---|---|---|
| View loans | `ui-users.loans.view` (Users: User loans view) | `Data - UI-Users Loans - View` — confirmed (C1621350 precondition "data - UI-Users Loans - view"; Cypress `uiUsersLoansView`) |
| Renew (also grants view since UIU-1140) | `ui-users.loans-renew.create` (Users: User loans renew) | `Data - UI-Users Loans-Renew - Create` — confirmed (C1405041; Cypress `uiUsersLoansRenew`) |
| View, change due date, renew | `ui-users.loans.all` (Users: User loans view, change due date, renew) | `Data - UI-Users Loans - Manage` (capability set name not confirmed — verify) |
| Change due date | `ui-users.loans-due-date.edit` (Users: User loans change due date) | `Data - UI-Users Loans-Due-Date - Edit` (capability set name not confirmed — verify) |
| Renew through override | `ui-users.loans-renew-override.create` (Users: User loans renew through override) | `Data - UI-Users Loans-Renew-Override - Create` (capability set name not confirmed — verify) |
| Add patron/staff info | `ui-users.loans-add-info.create` (Users: User loans: add staff or patron information) | `Data - UI-Users Loans-Add-Info - Create` (capability set name not confirmed — verify) |
| Declare lost | `ui-users.loans-declare-item-lost.execute` (Users: User loans declare lost) | `Procedural - UI-Users Loans-Declare-Item-Lost - Execute` (capability set name not confirmed — verify) |
| Claim returned | `ui-users.loans-claim-item-returned.execute` (Users: User loans claim returned) | `Procedural - UI-Users Loans-Claim-Item-Returned - Execute` (capability set name not confirmed — verify) |
| Mark claimed returned as missing | `ui-users.loans-declare-claimed-returned-item-as-missing.execute` (Users: User loans mark claimed returned missing) | `Procedural - UI-Users Loans-Declare-Claimed-Returned-Item-As-Missing - Execute` (capability set name not confirmed — verify) |
| Anonymize loans | `ui-users.loans-anonymize.execute` (Users: User loans anonymize) | `Procedural - UI-Users Loans-Anonymize - Execute` (capability set name not confirmed — verify) |
| Lost items requiring actual cost | `ui-users.lost-items-requiring-actual-cost.execute` | `Procedural - UI-Users Lost-Items-Requiring-Actual-Cost - Execute` (capability set name not confirmed — verify) |
| Circulation log (verification steps) | — | `Data - UI-Circulation-Log Log-Event - Manage` — confirmed (C1621350 precondition "data - UI-Circulation-Log Log-Event - manage"; not in the Cypress dictionary) |

> The previous build's names (`Procedural - UI-Users Loans Declare-Lost - Execute`, `… Claim-Returned …`, `… Renew-Override …`, `Data - UI-Users Loans Anonymize - Execute`) do not match the permission names in ui-users package.json and were replaced. Older cases still use legacy displayNames ("view loans, renew loans, renew through override") — prefer capability sets in new cases. TestRail P14 semantic search for the unconfirmed names (2026-11-25) returned no case precondition containing them.


---

## ECS / Multi-Tenant Notes

- ECS with TLR enabled: renewal skips circulation rules and reuses the loan's stored loan policy when `ecsTlrFeatureEnabled AND (central tenant OR (secure tenant AND not an oLOAN collection))` (CIRC-2344, Ramsons 24.3.18 / Trillium 24.5.0; CIRC-2565 Sunflower backport).
- Fee/fine workflows are not supported in the LoC Secure tenant — declaring lost a Congressional loan with a >0 set-cost policy is out of scope (CIRC-2673 closed with that comment).
- Duplicate `Aged to lost` actions / double billing were seen only in the ECS Professional tenant on Trillium BF (CIRC-2629) — assert exactly one `Aged to lost` row per loan in ECS cases.
- Loan/request data must reset when switching users or returning to Open loans (UIU-3460, UIU-3612 verified in non-ECS, ECS member, ECS central with mod-tlr).
- No `custom_ecs_enabled` cases exist in the Loans sections (TestRail suite 21 section 97 subtree, case fields as of 2026-11-24).

---

## Existing automated coverage

- **Cypress** (`folio-org/stripes-testing`, `cypress/e2e/loans/`): C561 loan-details, C562 loans-action-menu-options, C565 single-loan, C566 loan-dates, C567 renewal-success, C568 renewal-failure, C570 renewal-failure-due-date-schedule, C572 multiple-loans, C580 renewal-success-loan-details, C564 renewal-success-action-menu, C581 loan-change-due-dates, C721 export-loans, C779 export-overdue-loans, C3482 view-open-requests-on-loaned-items, C9192 loans-declared-lost, C9283 loan-details-fee-policy-links, C10949 lost-loans, C10958 loans-actions-while-claim-returned, C10959 loans-claim-returned, C10960 resolve-claim-returned, C15492 renewal-failure-not-loanable, C451617–C451620 view-large-*-loan-list, C1405041 loans-renew-capability-set, loan-policies.cy.js (C829891…C844878). Related: `circulation-log/filter-*` (renewed, renewed-through-override, declared-lost, claimed-returned, closed-loan), `fees-&-fines/claimed-returned-*`, `fees-&-fines/patron-block/*renewed*`. Fragments: `cypress/support/fragments/users/loans/{userLoans,renewConfirmationModal,overrideAndRenewModal,confirmClaimReturnedModal,confirmItemStatusModal}.js`, `fragments/loans/{loansPage,changeDueDateForm,renewals}.js`, `fragments/settings/circulation/loanAnonymization.js`.
- **Karate** (`folio-org/folio-integration-tests`, `mod-circulation/src/main/resources/vega/mod-circulation/features/`): `loans.feature` — declare lost (declaredLostDate, item status, billing), claim returned (suspend/refund lost fees), aged to lost (+ delayed billing), renewal (count/date), renew-by-barcode invalid/not allowed → 422 with error, override renewal without comment / due date → 422, overdue fine on renewal, patron-group limits blocking renewal; `loans-extended.feature` — scheduled anonymization with fee/fine branch, anonymization 1 hour after fine closes, actual-cost fee closed on check-in after policy change (CIRC-2130). Runners: `ModCirculationTests.java`, `ModCirculationExtendedTests.java`.
- Not automated (manual only): automatic renewals (C1621343–C1621440, `custom_automation_type` = manual), anonymization-through-settings matrix (C9218–C9235), additional loan comments (C397378/9), recall due-date recalculation (C466157–C466159).

---

## Common Verification Patterns

### Declare lost (Cancel then confirm)
1. Loan action menu on a Checked out item → `Declare lost` → modal `Confirm item status: Declared lost`; `Cancel` → status unchanged.
2. Re-open, enter text, `Confirm` → `Item status` = `Declared lost`; history row: Action `Declared lost`, Comments = text, Source = staff (linked).

### Status gates the action buttons
Assert the enabled/disabled matrix per item status — Declared lost: `Renew` active, `Change due date` + `Declare lost` inactive; Claimed returned: `Renew` + `Change due date` inactive, `Resolve claim` present; bulk `Change due date` stays inactive until a Checked-out loan is also selected (C9192, C10958).

### Change due date with a mixed selection
Checked out + Declared lost → bulk `Change due date` → Alert details `Item is Declared lost`; on save the checked-out loan gets the new due date, the lost loan shows `Failed to change due date: item is Declared lost` (C9192).

### Renewal failure → override
Renew → `Renew Confirmation` with `Item not renewed:` + reason → `Override` (only with override capability) → `Override & renew` → pick date or see `Due date will be calculated automatically` → comment → `Override` → history `Renewed through override` (C568, C9181).

### Bulk renew-override sends one request per loan (C1347104)
Override & renew on many loans fires requests one by one (UIU-3561 changelog: "Send bulk override renewal requests sequentially") — assert per-loan due date/status, and that an automated patron block (e.g. `Maximum number of lost items`) clears afterwards.

### Automatic renewal verification (C1621350, C1621420)
After the scheduled cycle: Open loans row shows new due date + `Renewal count`; Loan details has exactly one `Renewed` row with Source `System`; Circulation log (search by user/item barcode) shows one `Renewed` action (System source) and the renewal notice; ineligible loans show no `Renewed` entry. Preconditions require a persistent environment (not Snapshot).

---

## Key Business Rules for Test Cases

1. Loan actions are gated by item status: for a Declared-lost loan `Change due date` and `Declare lost` are unavailable and `Renew` stays available (override needed). (C9192)
2. `Declare lost` requires a free-text comment; it is stored in the history row `Comments`. (C9191; en.json `loans.declareLost.additionalInfoPlaceholder`)
3. Declaring lost sets `Item status = Declared lost`, fills the `Lost` field and disables `Declare lost`. (C9191)
4. Renewing a Declared-lost or Aged-to-lost item fails with `Item not renewed:` + `Item is Declared lost.` / `Item is Aged to lost.` and offers `Override`; override auto-calculates the due date and returns the item to `Checked out`. (C9181, C9192; en.json `messages.itemDeclaredLostNotRenewable`)
5. Bulk change due date skips ineligible loans with a per-row alert instead of failing the whole batch. (C9192, C10958; ssc `cddd.changeSucceededWithFailures`)
6. Claimed returned keeps the loan open and suspends lost-item fees already billed (refunds paid ones); the claim is resolved at check in (Found by library / Returned by patron) or via Resolve claim. (Karate `loans.feature` claim-returned scenario; docs Users "Mark an item claim returned"; C10960)
7. History `Source` records the acting staff user (Lastname, firstname middlename) linked to the profile; system actions show `System`. (C9191, C1621350; en.json `loans.action.source.system`)
8. Scheduled anonymization removes the patron link from closed loans per Settings > Circulation > Loan anonymization; anonymized loans disappear from the user's Closed loans. (C9218–C9235; en.json `anonymization.confirmation.message`)
9. Renewals honour the loan policy's renewable flag, renewal limit, renew-from and period; failures show the policy reason and may be overridable. (C568, C569, C575–C579)
10. Patron/staff info can be added to a loan and is logged in the Circulation log. (C397378, C397379; Cypress `patron-notices/correct-value-sent-on-loanadditionalinfo-token.cy.js`)
11. Bulk override renewal sends one request per loan sequentially, not a single batch. (C1347104; ui-users CHANGELOG 13.1.0 UIU-3561)
12. An automated patron block tied to loan state (e.g. max lost items) clears once the condition is resolved — assert the block's disappearance separately. (C1347104)
13. Claimed-returned loans replace `Claim returned` with `Resolve claim` (Declare lost / Mark as missing); either resolution resets `Claimed returned` to `-` and auto-creates a patron note. (C10960)
14. Mark as missing closes the loan and disables all loan-details buttons; Declare lost from Resolve claim keeps the loan open as Declared lost. (C10960)
15. Renewal failures show `Item not renewed:` plus a localized reason chosen by the backend error code; `Override` appears only for users with the renew-through-override capability. (C568, C569, C571, C6667; CIRC-2632, UIU-3537)
16. A recall recalculates due date: unchanged if (today + Recall return interval) is after the current due date; else the recall date, but never earlier than checkout date + Minimum guaranteed loan period. (C466157, C466158, C466159)
17. Manual anonymization is partial-success: loans with associated fees/fines are skipped and counted in `{amount} loan(s) had associated fees/fines and could not be anonymized.` (C9217, C422021; en.json `anonymization.error.message`)
18. New patron/staff info never overwrites: the previous entry is relabelled `Patron info (SUPERSEDED)` / `Staff info (SUPERSEDED)`. (C397378, C397379; en.json `data.loanActionMap.patronInfo.superseded`)
19. A lost item fee policy with $0 set cost and no processing fee for staff-declared lost can auto-close the loan after the configured interval → `Lost and paid` with two stacked rows. (C10948)
20. A backdated check-in sets the closed loan's `Return date` / `Check in service point` to the backdated values while history `Action Date` keeps system time. (C584)
21. Claimed-returned loans cannot be renewed or have their due date changed, and the renew failure modal shows NO `Override` button for them. (C9181, C10958)
22. Single-loan renewal success shows the callout `The loan for {title} was successfully renewed.`; multi-loan renewal opens the `Renew Confirmation` modal. (en.json `loans.item.renewed.callout`, `brd.renewConfirmation`; UIU-3625, C567)
23. A user holding only `Data - UI-Users Loans-Renew - Create` can open the patron's loans and renew (view was added to the renew set). (UIU-1140, C1405041)
24. Declare lost is single-loan only (no bulk); Claim returned works single or bulk, and bulk reports already-claimed items as `Item could not be claimed returned: item is already claimed returned`. (docs Users "Mark an item Declared lost"; en.json `bulkClaimReturned.status.notOk`)
25. Change due date warns but allows a past date, a date after patron expiration and a date when the library is closed. (docs Users "Change a due date"; ssc `cddd.warning.*`)
26. A loan whose item was deleted shows `-` for item fields and disables Renew / Claim returned / Change due date / Print due date receipt / Declare lost / Mark as missing, plus its Open-loans checkbox. (UIU-2531, C656307)
27. API: declare-item-lost requires `declaredLostDateTime` + `servicePointId`; claim-item-returned requires `itemClaimedReturnedDateTime`; mark-as-missing requires `comment`; add-info requires `action` (`staffInfoAdded`/`patronInfoAdded`) + `actionComment`; renew-by-barcode requires `itemBarcode` + `userBarcode`. (mod-circulation `ramls/*-request.json`)
28. Override renewal without a comment, or without a due date when one cannot be calculated, returns 422 and does not renew. (Karate `loans.feature`; codes `OVERRIDE_RENEWAL_COMMENT_REQUIRED`, `OVERRIDE_RENEWAL_DUE_DATE_REQUIRED`; C1528148)
29. Automatic renewal applies only when the loan policy enables it; it runs on the due date or on the configured days before it, then on retries. (C1621343, C1621350; CIRC-2584, CIRC-2687)
30. Automatic renewal is not attempted for short-term (interval under one day) and legacy-policy loans. (C1621419; ui-circulation `settings.loanPolicy.automaticRenewals.infoPopover`; CIRC-2687 "absent automaticRenewal = disabled")
31. Automatic renewal skips Aged to lost, Declared lost, Claimed returned items and closed loans (no `Renewed` entry in Circulation log). (C1621420)
32. Automatic renewal is blocked by an active hold or recall (failure logged in the Circulation log) and is not performed for inactive or blocked patrons. (C1621437, C1621440; CIRC-2584 NEWS 24.6.0)
33. Automatic renewal reloads the referenced loan policy but does not re-evaluate circulation rules for the existing loan, and schedules the next cycle from the new due date. (C1621354, C1621362, C1621352)
34. A successful automatic renewal adds one `Renewed` history row with Source `System`, increments `Renewal count`, and sends the "Item renewed" patron notice. (C1621350)
35. Lost-item fees billed on an Aged-to-lost item must be closed on check in even if the lost item fee policy switched between Set cost and Actual cost. (CIRC-2130; Karate `loans-extended.feature`)
36. Fees/fines of a previous loan of the same item stay actionable when the item is later checked out to another patron and claimed returned; only the new loan's lost fees are suspended. (UIU-3125)
37. Resolving a claim as Mark as missing should cancel the suspended lost-item fees; this was reported broken and closed Won't do (workaround: cancel fees manually) — do not assert automatic cancellation. (UIU-3490, UIU-3573; UIU-3544 draft)
38. With user-specific anonymization enabled, the user's `Anonymize closed loans` preference decides when closed loans are anonymized (library policy / earliest / never); manual anonymization is unaffected; preferences are resynced nightly. (UIU-3595, UICIRC-1361, CIRC-2698, CIRC-2699)
39. The Open loans list is sorted case-insensitively ascending (e.g. by call number). (UIU-3369, C1504467)
40. Open loans must show only the selected user's loans — no stale rows or counts from a previously viewed user. (UIU-3612, UIU-3460)
41. User-specific anonymization policy pick per closed loan: feature off or "According to library policy" or user deleted/not found → library policy; "As early as possible" → "earliest" policy (library policy if earliest is not configured); "Never" → loan is never anonymized. (Confluence "User-specific loan anonymization" page 2129002498, DRAFT; CIRC-2698)
42. "X after loan closes" counts from the system return date; loans closed without check-in (lost item paid for, claimed returned → marked missing) have none, so they are never anonymized under an interval rule but are anonymized under "Immediately"; a loan with an open fee/fine is not anonymized while fees/fines are treated differently. (Confluence page 2129002498; C9218–C9235)

---

## Authoring style (measured 2026-07-23)

Loans skews **`Other` (~85%, 59/69; TestRail S21 section 97 case-type counts)** with only ~15% `Functional` — median ~6 steps, `User Journey` ~0%. Most cases verify loan-details / closed-loan display and single actions as compact checks; end-to-end journeys live in Check out / Check in. Use `Other` for display/single-action cases; `Functional` for behaviour that computes state (recall recalculation, bulk renew/override, automatic renewal). Preconditions carry the loan setup with symbolic identifiers (`<barcode>`, service point `S`, loan policy with the relevant fields); automatic-renewal cases state "persistent environment — not Snapshot" (C1621350). New Umbrellaleaf cases carry capability sets in preconditions (C1405041, C1621350).

---

## Known Gaps / Items to Verify

- [ ] Declared-lost warning casing: C9192 asserts `Item is declared lost`; ssc `cddd.itemDeclaredLostWarning` = `Item is Declared lost`. Use the translation.
- [ ] Override auto due date text: C9192 asserts `New due date will be calculated automatically.`; en.json `override.autoNewDueDate` = `Due date will be calculated automatically`. Use the translation; C9192 needs an update.
- [ ] Claimed-returned renew reason: C10958 asserts `Item not renewed: Item is Claim returned`; en.json `messages.itemClaimedReturnedNotRenewable` = `Item is Claimed returned.`.
- [ ] Loan-closed action label: C10948 asserts `Loan closed`; en.json `data.loanActionMap.closedLoan` = `Closed loan`.
- [ ] Claim returned modal title: C10959 describes the modal by body text only; en.json uses `Confirm item status: Claimed returned` (single) and `Confirm claim returned` (bulk). Verify which one the single flow shows in env.
- [ ] `Due date & time has successfully been changed` (ssc `cddd.changeSucceeded`) — not confirmed whether it renders as a callout or inside the dialog.
- [ ] Nine capability set names (Loans - Manage, Loans-Due-Date, Loans-Renew-Override, Loans-Add-Info, Loans-Declare-Item-Lost, Loans-Claim-Item-Returned, Loans-Declare-Claimed-Returned-Item-As-Missing, Loans-Anonymize, Lost-Items-Requiring-Actual-Cost) are marked `(capability set name not confirmed — verify)`: they are built from permission names; neither the Cypress dictionary nor any TestRail precondition found confirms them — verify exact Eureka names in env.
- [ ] `Patron blocked from renewing`, `Reason for block:`, `View block details`, `Comment*` come only from C170386 (strings live outside ui-users en.json loans keys) — not re-verified against translations.
- [ ] Automatic renewal is still evolving: CIRC-2584 status Blocked (2026-11-24); retry semantics ("Attempt additional retries for") and the failure notice wording are not yet confirmed by Done stories.
- [ ] UIU-3595 / UICIRC-1361 (user-specific anonymization) are In QA — UI may still change; en_US.json does not yet contain the new keys.
- [ ] en_US.json lags en.json for 30 loans keys (all `messages.*` renewal reasons, `extended.loanAnonymizationPreference.*`) — non-English locales may still show backend English.
- [ ] Whether superseded patron/staff info remains individually searchable in the Circulation log — not re-verified.
- [ ] UIU-3561 (patron blocks out of sync after bulk renewal, optimistic locking) is still Open — bulk renew + block-clearing assertions may be flaky until it is fixed.
- [ ] Recall due-date rules (C466157–C466159) rely on cases only; no current Jira/Karate source was re-checked.
- [ ] Release-to-version mapping (Trillium = ui-users 13.0.x / mod-circulation 24.5.0; Sunflower = 12.x / 24.4.0; Ramsons = 11.x / 24.3.0) is inferred from CHANGELOG/NEWS dates; most UIU/CIRC Done issues have no fixVersion set.
- [ ] Confluence: only the DRAFT design page "User-specific loan anonymization" (2129002498) was used; no Confluence page for automatic renewals was found by title search.
