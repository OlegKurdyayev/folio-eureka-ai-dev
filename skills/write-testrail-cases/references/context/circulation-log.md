# FOLIO Circulation Log — Business Logic Context

> Built: 2026-07-21 · Refreshed: 2026-11-24 · Release: Umbrellaleaf (R2 2026) — ui-circulation-log master 6.2.0 (IN_PROGRESS; last release 6.1.0 = Trillium R1 2026, 2026-04-17), mod-audit master 3.1.0 (in progress; 3.0.0 = Trillium, 2026-04-16) · Builder: build-app-context (full refresh 2026-11-24; previous delta refresh 2026-11-07)

> Sources (2026-11-24 refresh): GitHub `folio-org/ui-circulation-log` master (`translations/ui-circulation-log/en.json` — identical to `en_US.json` at refresh time, `package.json`, `src/**`, `CHANGELOG.md`); `folio-org/mod-audit` master (`descriptors/ModuleDescriptor-template.json`, `ramls/circulation-logs.raml`, `ramls/logRecord.json`, `ramls/schemas/{logged_object_type,action_type}.json`, `mod-audit-server/src/main/java/org/folio/builder/description/*`, `NEWS.md`); `folio-org/folio-integration-tests` (`mod-audit/`); `folio-org/stripes-testing` (`cypress/e2e/circulation-log/`, `cypress/support/fragments/circulation-log/`); Jira UICIRCLOG + MODAUD resolved since 2024-06 (31 + 107 issues scanned, ~35 read in detail) plus cross-project issues mentioning "circulation log" (CIRC, MODEXPW, MODTLR, UXPROD …); TestRail project 14 / suite 21, section 1539 "Circulation log" (87 cases) + subsection 25716 "Additional Loan Comments" (2 cases) + C1621493 (Check in section).
> This is a small, single-purpose read-only audit app: it has no create/edit of its own — every row is produced by an action in another app (Check in, Check out, Requests, Loans, Fees/Fines, Users) and viewed/filtered here. Cases that assert "a notice/action was logged" should read the destination app's context file (check-in.md, check-out.md, requests.md, loans.md, fees-fines.md) for the triggering action, and this file for the exact log row shape and filter behavior.

---

## What is Circulation Log

The Circulation log app is a searchable, filterable audit trail of circulation events: every check-in, check-out, loan change, request status change, fee/fine action, notice, and patron/manual block gets one row here. Staff use it to investigate "what happened to this item/patron" without digging through individual records. It is read-only — the only actions available are jumping to the related record and exporting the result set to CSV. Data is stored and served by **mod-audit** (`GET /audit-data/circulation/logs`); events arrive from mod-circulation / mod-feesfines / mod-patron-blocks via Kafka `LOG_RECORD` events (PubSub removed — MODAUD-163, mod-audit 3.1.0 NEWS.md "Breaking changes").

---

## UI structure [confirmed]

- **Search & filter pane** (C1307995; `src/CirculationLogList/CirculationLogListFilter/*`): `Reset all` button (disabled while the URL has no filters or while loading — `CirculationLogList.js` `ResetButton disabled={!location.search || isLoading}`); one text-filter form with `User barcode` input + `Patron look-up` link below it, `Item barcode` input, `Description` input, and **one** `Apply` button under Description that submits all three text fields together (`TextFilters.js`; C958914); then accordions in this order: `Date` (**expanded** by default, `From`/`To` + `Apply`), `Service point` (**expanded** by default, multiselect of tenant service points — `closedByDefault={false}`), `Loan`, `Notice`, `Fee/fine`, `Request` (**collapsed** by default, each a checkbox list of circ actions). There are **no** filter accordions for Patron block / Manual block / Item block — those rows are reachable only by text filters, Date, or Service point (`CirculationLogListFilter.js`).
- **Empty start**: the app makes no API call until at least one filter is set (`useCirculationLog(false, …)` → `hasToCallAPI = filtersCount > 0`); opening the app shows no results (C402746 step 1 "The circulation app with no results is displayed").
- **Results table columns** (en.json `logEvent.*`): `User barcode`, `Item barcode`, `Object`, `Circ action`, `Date`, `Service point`, `Source`, `Description`, and `Action` (per-row ellipsis; only with `Circulation log: All`). Sortable: all except `Service point` and `Action`; default sort `Date` descending, and any column sort is applied with date-desc as the secondary sort (`CirculationLogList.js` `sortableFields`, `utils.js` `DATE_SORT_CLAUSE`).
- **Item barcode cell**: if a row has several items, only the first is shown with a `Show more` / `Show less` toggle (en.json `logEvent.item.showMore` / `showLess`; `CirculationLogEventItems.js`). The barcode is a link to the item record (`/inventory/view/{instanceId}/{holdingId}/{itemId}`) only when the user has `ui-inventory.item.edit` and the item is **not** a DCB virtual item; otherwise plain text (`CirculationLogEventItem.js`; UICIRCLOG-140, MODAUD-177).
- **Object `N/A`** renders as `-` (en.json `logEvent.object.N/A`).
- **Date column** is formatted in the browser locale as numeric date + hour:minute (`FormattedTime.js`).
- **Pagination** (C1307995; `CirculationLogListContainer.js` `RESULT_COUNT_INCREMENT = 100`): 100 rows per page; `Previous`/`Next` with the current row range between them; `Previous` disabled on the first page. Applying any new filter **resets offset to 0** (`useLocationFilters.js`; UICIRCLOG-185, released in 6.1.0).
- **Results counter**: the total comes from a separate `limit=0` call to get the real `totalRecords` (UICIRCLOG-179, 6.0.2) — this is what export parity is checked against (C825329).
- **Per-row `Action` ellipsis menu** — see "Action menu matrix" below. `Item details` is **not** an ellipsis option in the current code: item navigation is the barcode link (see Known Gaps).
- Reset: `Reset all` clears every filter (URL search emptied). Filters live in the URL, so returning to the app via its icon restores the last filtered view (C16979, C16981, C16995, C16998).

---

## API surface (mod-audit, `circulation-logs` interface 1.3)

| Method | Path | permissionsRequired | Notes |
|---|---|---|---|
| GET | `/audit-data/circulation/logs` | `circulation-logs.collection.get` | CQL `query`, `limit`, `offset`. UI default query `cql.allRecords=1 sortby date/sort.descending`. (`descriptors/ModuleDescriptor-template.json`, `ramls/circulation-logs.raml`) |
| POST | `/data-export-spring/jobs` (mod-data-export-spring) | `data-export.job.item.post` | Export: body `{"type":"CIRCULATION_LOG","exportTypeSpecificParameters":{"query":…}}` (`useCirculationLogExport.js`) |
| GET | `/data-export-spring/jobs/{id}` and `/{id}/download` | `data-export.job.item.get` / `data-export.job.item.download` | UI polls every 10 s until `SUCCESSFUL`/`FAILED`, then auto-downloads `fileNames[0]` (`useCirculationLogExport.js`) |

UI filter → CQL mapping (`src/CirculationLogList/constants.js` `CUSTOM_FILTERS`, `utils.js`):
- `User barcode` → `userBarcode=="*value*"` — **contains** match, so `123` also returns `12345` (UICIRCLOG-139, converted to task, not changed).
- `Item barcode` → `items="value"` — **exact** match (MODAUD-252 comment: right-truncated barcode search is not supported/slow).
- `Description` → `description=="*value*"` — contains match on the log row's Description, **not** on the source record's own description (UICIRCLOG-156 closed as not-a-bug).
- `Date` → `date>="<From 00:00 in tenant TZ as ISO>" and date<="<To 23:59:59.999 in tenant TZ as ISO>"` (UICIRCLOG-125; `buildDatesWithTimeZoneOffsets`).
- All checked circ-action checkboxes across Loan/Notice/Fee/fine/Request are merged into one `action` list (OR); text/date/service-point filters are ANDed with it.

Log record fields (`ramls/logRecord.json`): `id`, `eventId`, `userBarcode`, `items[]` (itemBarcode, itemId, instanceId, holdingId, loanId), `object`, `action`, `date`, `servicePointId`, `source`, `description`, `linkToIds` (`userId`, `requestId`, `feeFineId`, `templateId`, `noticePolicyId`).
- `object` enum: `N/A`, `Fee/Fine`, `Item Block`, `Loan`, `Manual Block`, `Notice`, `Patron Block`, `Request`, `CheckIn/CheckOut` (`schemas/logged_object_type.json`).

---

## Exact UI Texts [from `en.json`, keys named]

### Circ action values (stored action → displayed label, en.json `logEvent.action.<stored>`)

| Object | Circ action values (displayed) |
|---|---|
| Loan | `Checked out`, `Checked out through override`, `Checked in`, `Renewed`, `Renewed through override`, `Changed due date`, `Claimed returned`, `Declared lost`, `Aged to lost` (stored `Age to lost`), `Anonymized` (stored `Anonymize`), `Closed loan`, `Marked as missing`, `Recall requested`, `Patron info added`, `Staff info added`, `Patron info (SUPERSEDED)` (key `Patron info superseded`), `In use` (stored `Picked up for use at location`), `Held` (stored `Held for use at location`) |
| Request | `Created`, `Created through override`, `Cancelled`, `Moved`, `Queue position reordered`, `Request status changed`, `Expired`, `Pickup expired` |
| Fee/fine | `Billed`, `Paid fully`, `Paid partially`, `Waived fully`, `Waived partially`, `Transferred fully`, `Transferred partially`, `Refunded fully`, `Refunded partially`, `Credited fully`, `Cancelled as error`, `Staff information only added` (C17047) |
| Patron block | `Patron blocked from requesting` |
| Manual block | `Created`, `Edited`, `Deleted` |
| Item block | `Created`, `Edited`, `Deleted` |
| Notice | `Send`, `Send error` |

> Also defined in en.json (no filter checkbox): `Check in`, `Check out`, `Modified`. `Recall requested` is a **Loan**-accordion checkbox (`LOAN_ACTIONS.RECALL_REQUEST`), not a Request one. Object labels (en.json `logEvent.object.*`): `Fee/fine`, `Item block`, `Loan`, `Manual block`, `Notice`, `Patron block`, `Request`, `-`.
> `Patron info (SUPERSEDED)` is computed in the UI, not stored: for each item barcode only the most recent `Patron info added` row keeps that label; every older one on the same item displays `Patron info (SUPERSEDED)` (`markOldPatronInfoAsSuperseded.js`; C402746 "The older patron info will display as patron info (superseded)").

### Filter checkbox options per accordion (`src/CirculationLogList/constants.js`, order as in code)

| Accordion | Checkboxes |
|---|---|
| `Loan` (17) | Changed due date, Patron info added, Staff info added, Checked out, Checked out through override, Checked in, Anonymized, Claimed returned, Closed loan, Declared lost, Marked as missing, Recall requested, Renewed, Renewed through override, Aged to lost, In use, Held |
| `Notice` (2) | Send, Send error |
| `Fee/fine` (12) | Billed, Credited fully, Paid fully, Paid partially, Refunded fully, Refunded partially, Staff information only added, Transferred fully, Transferred partially, Waived fully, Waived partially, Cancelled as error |
| `Request` (6) | Cancelled, Created, Pickup expired, Expired, Moved, Queue position reordered |

> No checkbox for: `Request status changed`, `Created through override`, `Patron blocked from requesting`, block actions. To find those rows use barcode/Description/Date/Service point filters.

### Action menu matrix (`src/CirculationLogList/CirculationLogEventActions/{utils,CirculationLogEventActions}.js`)

Menu order when present: `Fee/fine details` → `Loan details` → `User details` → `Request details` → `Notice policy` → `Live version of template` (en.json `logEvent.actions.*`). Each option also needs the viewer's permission in the other app; if none qualifies the ellipsis is not rendered.

| Option | Shown for Object | Data condition | Extra permission (legacy name) | Target |
|---|---|---|---|---|
| `Fee/fine details` | Fee/Fine, Notice | `linkToIds.feeFineId` + `userId` | `ui-users.feesfines.actions.all` | `/users/{userId}/accounts/view/{feeFineId}` |
| `Loan details` | Loan, Fee/Fine, Request, Notice | `userId` and exactly **one** item with `loanId` | `ui-users.loans.view` | `/users/{userId}/loans/view/{loanId}` |
| `User details` | Loan, Fee/Fine, Request, Notice, Manual Block | `userId` | `ui-users.view` | `/users/view/{userId}` |
| `Request details` | Loan, Fee/Fine, Request, Notice | `requestId` | `ui-users.requests.all` | `/requests/view/{requestId}` |
| `Notice policy` | Notice | `noticePolicyId` | `ui-circulation.settings.notice-policies` | `/settings/circulation/notice-policies/{id}` |
| `Live version of template` | Notice | `templateId` | `ui-circulation.settings.notice-templates` | `/settings/circulation/patron-notices/{id}` |

Patron block and Item block rows never get an ellipsis menu; anonymized rows (no `userId`) lose `User details`/`Loan details`.

### Description formats (mod-audit `builder/description/*`)

| Event | Description pattern (literal text) | Source |
|---|---|---|
| Check out | `Checked out to proxy: {proxyBarcode or "no"}.` | `Descriptions.CHECKED_OUT_TO_PROXY_MSG` |
| Check in (no loan) | `Item status: {status}.` / In transit: `Item status: In transit to {destination SP}.` | `ItemCheckInDescriptionBuilder.java`; C1621493 |
| Check in / Closed loan (loan) | `Item status: {status}` + optional `. Claimed returned item: {resolution}` + optional `. Backdated to: {yyyy-MM-dd HH:mm:ss}` (only when return date ≠ system return date, compared to the minute) + optional `. Overdue due date: {yyyy-MM-dd HH:mm:ss}` + `.`; dates in tenant time zone | `LoanCheckInDescriptionBuilder.java`; MODAUD-176, MODAUD-264 |
| Fee/fine billed | `Fee/Fine type: {type}. Fee/Fine owner: {owner}. Amount: {0.00}. {…}. Additional information to staff: {…}` | `Descriptions.BILLED_MSG` |
| Fee/fine paid / waived / refunded / transferred | `Fee/Fine type: {type}. Amount: {0.00}. Balance: {0.00}. {Payment method / Waive reason / Refund reason / Transfer account}: {…}. Additional information to staff: {…}. Additional information to patron: {…}.` | `Descriptions.FEE_FINE_*_MSG` |
| Fee/fine cancelled as error | `Amount: {0.00}. Cancellation reason: {…}. Additional information to patron: {…}.` | `Descriptions.CANCELLED_MSG` |
| Notice sent | `Template: {template}. Triggering event: {event}.` | `Descriptions.NOTICE_MSG` |
| Request created | `Type: {type}. Service point: {sp}. Address type: {…}. ` (present parts only) | `RequestDescriptionBuilder.java` |
| Request edited | `New request type: X (from: Y).` / `New expiration date: X (from: Y). ` / `New fulfilment preference: …` / `New pickup service point: …` / `New address type: …` | `RequestDescriptionBuilder.java` |
| Request moved | `New item barcode: X (from: Y).` | `RequestDescriptionBuilder.java` |
| Queue position reordered | `New queue position: X (from: Y).` | `RequestDescriptionBuilder.java` |
| Request status changed | `Type: … Pickup service point: … New request status: X (from: Y).` | `RequestStatusChangedDescriptionBuilder.java` |
| Request cancelled | `Reason for cancellation: {reason's Description (internal)}.` | `RequestDescriptionBuilder.java`; C1282808, MODAUD-317 |
| Manual block | `Description: …` `Staff only information: …` `Message to patron: …` `Expiration date: …` `Block actions: borrowing, renewals, requests. ` (present parts only) | `ManualBlockDescriptionBuilder.java` |

### Permissions [confirmed — `package.json` stripes.permissionSets, en.json `permission.*`]

| Display name | Permission | Sub-permissions |
|---|---|---|
| `Circulation log: View` | `ui-circulation-log.log-event.view` | `module.circulation-log.enabled` (incl. `stripes-core.settings.read` — UICIRCLOG-190), `inventory-storage.service-points.collection.get`, `circulation-logs.collection.get`, `data-export.job.item.get` |
| `Circulation log: All` | `ui-circulation-log.log-event.all` | `…log-event.view` + `data-export.job.item.post`, `data-export.job.item.download` (UICIRCLOG-148), `usergroups.collection.get`, `users.collection.get` (UICIRCLOG-167 — Patron look-up) |

### Export

`Export results (CSV)` (en.json `logEvents.actions.export`) in the results-pane `Actions` menu (menu exists only when the `data-export-spring` interface is present). Button is **disabled** when there are 0 results, while a request is in flight, or without `Circulation log: All` (`CirculationLogListActions.js`). Toasts:
- `Your Circulation log export has been requested. Please wait while the file is downloaded.` — job accepted (`logEvents.actions.export.success`)
- `Export job has been completed.` — job SUCCESSFUL, file then auto-downloads (`logEvents.actions.export.successful`)
- `Export job has been failed.` — job FAILED, error toast (`logEvents.actions.export.failed`)
- `Something went wrong with export file request, please try again.` — POST failed, error toast (`logEvents.actions.export.error`)
The job also appears in Export Manager with job type `Circulation log` (C353955).

---

## Capability Sets (Eureka)

| Action | Capability set (from TestRail preconditions) | Legacy equivalent |
|---|---|---|
| View / filter the log | `data - UI-Circulation-Log Log-Event - view` (C1282809, C1282810) | `Circulation log: View` |
| View + row actions + CSV export + Patron look-up | `data - UI-Circulation-Log Log-Event - manage` (C411752, C380404) | `Circulation log: All` |

Destination links additionally need the other app's capabilities (Users view, Loans view, Fee/fine actions, Requests, Circulation settings notice policies/templates — see Action menu matrix). Item barcode link needs Inventory item **edit** (`ui-inventory.item.edit`).

---

## Common Verification Patterns

### Filter by identifier / action, then verify the row (C15484, C16978, C16980, C16982, C16997, C16999, C17136)

1. Perform the triggering action in its own app (e.g. check out an item) — note the item/user barcode.
2. Open Circulation log; filter by the relevant identifier (`Item barcode` / `User barcode` / `Description`) OR by expanding the matching accordion (e.g. `Loan`) and checking the specific circ-action checkbox.
3. Expected: the row appears with the correct `Circ action` value and the populated columns (`User barcode`, `Item barcode`, `Object`, `Circ action`, `Date`, `Service point`, `Source`, `Description`).
4. `Reset all`, then re-find the same row via a different filter (barcode instead of checkbox) — the team routinely verifies **both** the checkbox-filter path and the barcode-filter path reach the same row.

### Cover every destination link (team convention — same "every entry point" principle as fees-fines)

Real cases (C16975, C16979, C16981, C16995, C16998, C17093) open **every applicable destination** one at a time and confirm the log view is preserved on return:
- Item barcode text link → item record loads (regression source: UIIN-2606, "Item barcode redirects to the broken page in circ log").
- Ellipsis `User details` / `Loan details` / `Request details` / `Fee/fine details` / `Notice policy` / `Live version of template` per the Action menu matrix.
- Return via the Circulation log app icon → same filtered log.
Don't assert options the matrix excludes for that Object (e.g. no `Loan details` on a multi-item row or on a Manual block row).

### Date range filter (C16976, C411752)

`From`/`To` date pickers + `Apply`. Team sequence: `From=today` alone → widen `From=yesterday` → narrow `To=yesterday` → clear via the filter chip `x`. With a non-UTC tenant time zone (e.g. America/Vancouver), a same-day filter returns only actions on that date **in tenant time**, and the CSV export has the same rows with tenant-time dates (C411752; UICIRCLOG-125, MODEXPW-613).

### Notice row navigation (C17092, C17093)

A Notice row's ellipsis offers `Loan details`, `User details`, `Notice policy`, `Live version of template` (plus `Fee/fine details`/`Request details` when the notice references them). A sent notice is findable both via `Notice` > `Send` and via the recipient's `User barcode`.

### CSV Export — record-count parity and ECS tenant isolation (C353955, C825329)

- Exported row count (lines minus 1 header) equals the on-screen counter and the API `totalRecords` from `GET /audit-data/circulation/logs?limit=0`, across filter combinations at ~100,000-record scale (UICIRCLOG-179, MODEXPW-607, MODEXPW-617/618). Open the CSV in a proper parser: MODEXPW-617 was closed after "missing" rows turned out to be an Excel display artefact.
- In ECS, an export from Tenant_1 is visible only in Tenant_1's Export Manager (`Circulation log` job type) and contains only Tenant_1 data.

### Permission tier — `Circulation log: View` (C365625; code)

Filters work; the whole `Action` column is absent (`CirculationLogList.js` `isActionColumnVisible = hasPerm('ui-circulation-log.log-event.all')`); the pane `Actions` menu opens but `Export results (CSV)` is shown **disabled** (`CirculationLogListActions.js`; Cypress `searchPane.checkExportResultIsUnavailable()` asserts `Button('Export results (CSV)', { disabled: true }).exists()`).

### Anonymization (C449382, C449383, C1543989, C17136)

Before anonymization the `Checked out` / `Closed loan` / `Checked in` rows carry the patron barcode. After "Anonymize all loans" on that closed loan the same rows (now findable by item barcode only) have no patron barcode, and a later check-in of the same item (home or other service point) also logs without a user barcode. `Anonymized` rows show Item barcode, Object, Date, Service point, Source (C17136). Applies to ECS loans too (C1543989).

### Source = acting user (C407706, C1259771, C1259786)

User A checks out; User B renews / changes due date → the `Renewed` and the companion `Changed due date` rows show Source = User B; the `Checked out` row stays User A (CIRC-2550 — both rows were fixed; also ECS member/central).

### Shared cancellation reason (C1282808, C1282809, C1282810)

Cancelling with a Consortium-Manager **Shared** reason logs `Request` / `Cancelled` with Description `Reason for cancellation: {Description (internal)}` in the tenant where the request was cancelled. For ECS requests the other tenant's row shows `Reason for cancellation: DCB Cancelled.` (Central request → Member row; Secure mediated request → Central and Member rows) — expected DCB behaviour (MODTLR-312 comment); may change with MODAUD-317.

### In-house use of a checked-out item (C1621493; CIRC-2691)

Check in with `In-house use` + confirm → Circulation log for the item shows two `Checked in` rows and one `Closed loan`: the first `Checked in` has the patron barcode, Description `Item status: Available.`; the second (in-house use) has User barcode `-` and the same Description; Source = logged-in user on both.

### Patron look-up and free-text filters (C958914)

`Patron look-up` opens the "Select User" modal; picking a user writes their barcode into `User barcode`. All three text fields have their own clear `x`; the single `Apply` submits all of them.

### `-` placeholder for barcode-less users (C360554)

A logged action whose user has no barcode shows `-` in `User barcode`.

### Description timestamps follow tenant time zone (C770448–C770456, C1554373/74/83, C1554431)

Dates inside Description (Changed due date, Closed loan, Recall requested, Renewed, Aged to lost) render in the tenant's Language-and-localization time zone, identically in the UI and the CSV (CIRC-2295, MODAUD-264).

---

## ECS / Multi-Tenant Notes

- Each tenant has its own log; there is no cross-tenant view. A cross-tenant request/loan produces rows in every tenant involved (C1282809, C1282810, C1259786, C1554374, C1554431).
- ECS cases use affiliation switching between Central / Member (College, University) / Secure; give `data - UI-Circulation-Log Log-Event - view` in every tenant checked (C1282809, C1282810).
- Export is strictly per tenant (C353955).
- Intermittent missing `Cancelled` rows in Central for ECS TLR cancellations (CIRC-2563) were not reproducible on snapshot and closed — keep as a regression scenario, not as expected behaviour.

---

## Existing automated coverage

- **Karate** — `folio-integration-tests/mod-audit/src/main/resources/vega/mod-audit/features/` (`ModAuditApiTest.java`): `checkInCheckOutEvent.feature` (check-out → +1 row; check-out + check-in closing loan → +3 rows; invalid user/item barcode → 0 new rows), `loanEvent.feature` (closed loan → last action `Closed loan`; renew → +2 rows, see CIRC-1165), `requestEvent.feature` (create/update request → +1 row; invalid itemId/requesterId → 0 rows). Endpoint covered: `GET audit-data/circulation/logs`. No Karate for export, filters/CQL, fee/fine, notice or block events.
- **Cypress** — `stripes-testing/cypress/e2e/circulation-log/*.cy.js` (27 specs) automate C15483–C17137 filter + "Check the Actions button" pairs, C350710–C350712 date/time, C358981, C360106, C360553, C360554, C365625, C368490, C407706, C645–C647, C770448, C770456; fragments `cypress/support/fragments/circulation-log/{searchPane,searchResults}.js`; FSE smoke `cypress/e2e/fse/{api,ui}/circulation-log-*.cy.js`.
- **Manual only** (no spec found): C1307995 pagination, C825329/C353955 export parity, C411752, C449382/83, C1543989, C1259771/86, C1282808–10, C958914, C380404, C770449/50/55, C1554373/74/83/431, C1621493, C402743/46.

---

## Key Business Rules for Test Cases

1. Circulation log has no create/edit of its own — a log-verification case's precondition is "perform action X in app Y". (cases)
2. The same event must be findable both by free-text identifier and by the circ-action checkbox. (C16978, C16980, C16982)
3. The ellipsis menu options depend on Object + linked IDs + the viewer's other-app permissions (see Action menu matrix); item navigation is the barcode link, not an ellipsis option. (`CirculationLogEventActions/utils.js`, `CirculationLogEventItem.js`)
4. Returning via the Circulation log app icon restores the last-applied filters/results (filters are kept in the URL). (C16979, C16981, C16995, C16998; `useLocationFilters.js`)
5. `Description` filter is a contains-match on the log row's Description (e.g. claimed-returned comment, patron/staff info), not on the source record's own description field. (C15853, UICIRCLOG-156)
6. Closed-loan rows must not show a spurious `Backdated to:`; it is added only when the return date differs from the system return date (compared to the minute). (C16999, MODAUD-176, `LoanCheckInDescriptionBuilder.java`)
7. `Export results (CSV)` is async: "requested" toast first, then `Export job has been completed.` + auto-download or `Export job has been failed.`; the UI polls the job every 10 s. (en.json, `useCirculationLogExport.js`)
8. Item-barcode link and the ellipsis links are separate entry points and have shipped bugs independently. (UIIN-2606)
9. 100 rows per page; applying any new filter resets the offset to page 1. (C1307995, UICIRCLOG-185)
10. Notice rows offer Loan/User/Notice policy/Live version of template (+ Fee/fine/Request details when referenced). (C17092, C17093, `utils.js`)
11. Exported CSV row count equals the on-screen counter and API `totalRecords`; ECS exports are per tenant. (C353955, C825329, MODEXPW-607)
12. `Circulation log: View` removes the entire Action column; `Export results (CSV)` stays in the Actions menu but disabled. (C365625, `CirculationLogList.js`, `CirculationLogListActions.js`, Cypress `searchPane.js`)
13. Anonymizing a loan strips patron identity from its historical rows and from later events on the same item. (C449382, C449383, C1543989)
14. Each row's Source is the user who performed that action; on renewal both the `Renewed` and the companion `Changed due date` rows show the renewing user. (C407706, C1259771, CIRC-2550)
15. A cancelled request's Description is `Reason for cancellation: {Description (internal)}`; for ECS requests the counterpart tenant shows `DCB Cancelled.` (C1282808, C1282809, C1282810)
16. A user with no barcode logs as `-` in User barcode. (C360554)
17. The app sends no search until at least one filter is applied; first open shows no results. (`useCirculationLog.js`, C402746)
18. `User barcode` filter is a contains match (`123` also finds `12345`); `Item barcode` filter is exact. (`constants.js` `CUSTOM_FILTERS`, UICIRCLOG-139, MODAUD-252)
19. Date filter boundaries are start/end of day in the tenant time zone, not UTC; the export uses the same boundaries. (C411752, UICIRCLOG-125, `utils.js`)
20. Multiple checked circ actions (even across accordions) are ORed; text, date and service-point filters are ANDed with them. (`utils.js` `buildLogEventsQuery`)
21. Loan accordion includes `In use` and `Held` (reading-room "for use at location" pickup/hold) since Trillium. (UICIRCLOG-180, UICIRCLOG-181, MODAUD-261)
22. Only the latest `Patron info added` per item keeps that label; older ones display `Patron info (SUPERSEDED)`. (C402746, `markOldPatronInfoAsSuperseded.js`)
23. Patron block, Manual block and Item block rows have no filter accordion; Manual block rows offer only `User details`; Patron/Item block rows have no ellipsis. (`CirculationLogListFilter.js`, `CirculationLogEventActions/utils.js`)
24. Item barcode is not a link for DCB virtual items or when the viewer lacks Inventory item edit permission. (`CirculationLogEventItem.js`, UICIRCLOG-140, MODAUD-177)
25. `Loan details` appears only when the row has a user and exactly one item with a loan ID. (`getHasLoanDetails`)
26. `Export results (CSV)` is disabled when the result count is 0. (`CirculationLogListActions.js`)
27. A check-in of an item not on loan has no user barcode (expected — CIRC-2390); the next check-in after a new loan has it again.
28. Checking in a checked-out item with in-house use logs two `Checked in` rows (the second with User barcode `-`) and one `Closed loan`. (C1621493, CIRC-2691)
29. Check-out + check-in that closes the loan adds 3 log rows (Checked out, Checked in, Closed loan); a renewal adds 2 rows; a failed action (invalid barcode/ID) adds none. (Karate `checkInCheckOutEvent.feature`, `loanEvent.feature`, `requestEvent.feature`)
30. Description dates (Changed due date, Closed loan, Recall requested, Renewed, Aged to lost) are rendered in tenant time zone in UI and CSV. (C770448–C770456, CIRC-2295, MODAUD-264)
31. The Service point column shows the service point the acting user had selected when performing the action. (C380404)
32. Columns are sortable except Service point and Action; any sort keeps date-descending as tie-breaker. (`CirculationLogList.js`, `utils.js`)

---

## Authoring style (measured 2026-07-23)

Circulation Log: **`Other` ~82%** / Func ~18%, median **~5 steps**, `User Journey` ~8%. Cases perform a circulation action (check out/in, request, fee/fine, notice) then verify the resulting **log entry** — correct action name, date, user, item, and that clicking through opens the right record. Keep them compact and `Other`; the value is asserting the exact logged action value and its link target, not re-testing the action itself. Preconditions set up the action's prerequisites; the log assertion is the point. Newer cases list both legacy permission and Eureka capability set in preconditions (C411752, C1282809).

---

## Known Gaps / Items to Verify

- [ ] Eureka capability-set spelling: most cases use `data - UI-Circulation-Log Log-Event - view|manage` (C1282809, C411752, C380404) but C1621493 uses `data - UI-Circulation-Log - View`. Verify in Settings > Authorization roles.
- [ ] `Item details` (en.json `logEvent.actions.itemDetails`) is still a translation key, but `CirculationLogEventActions.js` doesn't render it. Older cases (C16979) and the previous version of this file listed it as an ellipsis option. Verify in env whether the ellipsis shows it; if not, item navigation is the barcode link only.
- [ ] C365625 wording ("Export results (CSV)" absent) vs code/Cypress (present but disabled) — this file follows the code; ask the case owner to align C365625.
- [ ] Request cancellation Description: today it logs the reason's `Description (internal)` and literally `null` when that field is blank. MODAUD-317 (Open) proposes logging the reason **Name** instead. Re-check C1282808–C1282810 when MODAUD-317 is closed.
- [ ] `Overdue due date:` part of the check-in Description: in code it is appended when due date is **after** return date (`LoanCheckInDescriptionBuilder.java`), and CIRC-2295 asks to rename it to "Due date". Don't assert its presence/absence without checking in env.
- [ ] Request anonymization (CIRC-2292, mod-circulation 24.5.0) says logs get `userBarcode "-"` and a new `anonymizeRequest` action. That action is not in the mod-audit `action_type.json` enum or en.json, so the display is unknown.
- [ ] In-house use events as their own log action (UXPROD-5877, UXPROD-5885 "Umbrellaleaf (TBD)"): no new action key in en.json/mod-audit yet. Today in-house use shows up only as a plain `Checked in` row (C1621493).
- [ ] Kafka migration (MODAUD-163, mod-audit 3.1.0 breaking change) is unreleased in master. Regression risk is that no rows appear at all (MODAUD-323) or that fee/fine rows go missing (MODAUD-324, both seen during this work). Smoke one action per Object after upgrade.
- [ ] Actual-cost lost-item actions (UICIRCLOG-118/119/120/121 — "Actual cost (billed/cancelled/expired)") were closed as "Covered by Lists" — they are **not** circulation-log actions; don't write log cases for them.
- [ ] Exact CSV header names not taken from a real downloaded file (mod-data-export-worker not cloned in this refresh).
- [ ] The Actions "..." icon border-less styling (C368490) is a minor visual-only assertion. It's listed here only so nobody rediscovers it as "new".

---

## Changes since 2026-07-21 (delta refresh 2026-11-07, verified 2026-11-24)

> Folded into the sections above on 2026-11-24. Kept for traceability.
> Sources: GitHub folio-org/ui-circulation-log (`translations/ui-circulation-log/en.json`; **master is 6.2.0 IN_PROGRESS, last release 6.1.0** — the 2026-11-07 note "HEAD v6.0.0" was wrong per `package.json`/`CHANGELOG.md`); Jira Done issues; TestRail suite 21.

### What changed
- Fee/fine actions were missing from the Circulation log (MODAUD-324), and logs did not appear at all (MODAUD-323). Both were fixed on snapshot; they are now regression candidates (see Known Gaps, Kafka migration).
- 6.1.0 (Trillium): `In use` / `Held` Loan filter options (UICIRCLOG-180/181), offset reset on new search (UICIRCLOG-185), `stripes-core.settings.read` added to app permission (UICIRCLOG-190).
- mod-audit 3.0.0 (Trillium): Description dates in tenant time zone (MODAUD-264), shared cancellation reason logged in ECS (MODAUD-271), item-barcode search performance (MODAUD-252), use-at-location actions (MODAUD-261).
- CIRC-2550: Source of `Renewed` and companion `Changed due date` rows = acting user (C1259771, C1259786).
- CIRC-2691: in-house use of a checked-out item → two check-in rows (C1621493, created 2026-11).

### TestRail — cases created since 2026-07-21 relevant to this area

- C1621493 Circulation log shows separate check-in entries for the return and the in-house use of a checked-out item — refs CIRC-2691 (Check in section 115)
- C1554431 / C1554383 / C1554374 / C1554373 [LoC] "Description" field displays time according to current time zone … — refs CIRC-2295, MCBFF-218
- C1543989 [LoC] Verify that patron information is not stored in Circulation log after ECS loans anonymization — refs CIRC-2045, MCBFF-217
