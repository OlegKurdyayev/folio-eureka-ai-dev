# FOLIO Requests App — Business Logic Context

> Built: 2026-06-17 (Cypress-fragment refresh 2026-08-06) · Refreshed: 2026-11-24 (full source refresh, delta of 2026-11-07 folded in) · Release: Umbrellaleaf R2 2026 (ui-requests master 13.1.0 IN_PROGRESS; mod-circulation 24.6.0-SNAPSHOT; last released: Trillium R1 2026 = ui-requests 13.0.x) · Builder: build-app-context (Refresh Mode)

> **Refresh 2026-11-24 sources:** GitHub folio-org/ui-requests master (`translations/ui-requests/en.json` — identical to `en_US.json`, 306 keys; `package.json` stripes.permissionSets; `CHANGELOG.md`; `src/constants.js`, `src/utils.js`, `src/routes/RequestsRoute.js`); folio-org/mod-circulation master (`descriptors/ModuleDescriptor-template.json`, `ramls/request.json`); Jira UIREQ (85 Done/Closed stories+bugs resolved ≥ 2024-06-01, 49 with resolution Done) + CIRC (45 request-related Done) + UAL/anonymization issues; folio-integration-tests (`mod-circulation`, `folio-ecs-circulation`, `edge-patron`); stripes-testing (`cypress/e2e/requests`, `cypress/e2e/title-level-request`, `cypress/e2e/consortia/requests`, `cypress/support/fragments/requests/`); docs.folio.org source `content/en/docs/Access/Requests/requests.md`; Confluence pages "Request anonymization" (1056210996) and "Circulation - For use at location - Trillium - Notes on functionality" (1779171329). **TestRail was not queried in this refresh** (bridge disconnected) — case IDs come from the earlier build or stripes-testing spec titles.
> Release mapping (Jira UIREQ versions): Ramsons R2 2024 = ui-requests 11.x · Sunflower R1 2025 = 12.x · Trillium R1 2026 = 13.0.x · Umbrellaleaf R2 2026 = 13.1.0 (in progress).

> Generated/refreshed by the build-app-context workflow on 2026-06-17.
> **GitHub fragment refresh on 2026-08-06**: `stripes-testing/cypress/support/fragments/requests/requests.js` scanned. New items added: `# Copies` column, `Printed` column, `Export expired holds to CSV` action. All other strings confirmed current.
> Sources:
> - TestRail section subtree rooted at section_id=96 (suite_id=21): sections 96, 343, 870, 1134, 1527, 2013, 17479, 29720, 93820, 34771
> - TestRail coverage: 162 cases total in subtree; 44 cases tagged to focus releases Umbrellaleaf, Trillium, Sunflower, Ramsons
> - GitHub signal scan: folio-org/ui-requests plus public repo metadata for requests-related modules
> - (2026-06-17) Jira/component signal was not refreshed in that run — superseded by the 2026-11-24 refresh above
> Strings marked [from cases] are mined from TestRail corpus.
> Strings marked [from source, verify in env] come from docs or limited source signal and should be validated in the target tenant UI before strict assertions.

---

## What is Requests

The Requests app allows library staff to create and manage patron requests for items. FOLIO supports item-level requests on a specific item and title-level requests on an instance, where the system selects the item that should satisfy the request.

Requests drive the retrieval, transit, hold shelf, delivery, and fulfillment workflows. The app also exposes queue management, request notices, CSV exports, and request-printing flows that are reflected in the existing TestRail corpus.

> **Identifier convention for this area:** write circulation cases with **symbolic placeholders the executor fills at run time** — `service point S` / `S1`, `<barcode>`, `<title>`, `<requester>` — not invented concrete values. Do NOT fabricate a specific barcode or a named service point; concrete values only when the exact value is itself under test.

---

## Key Terms

| Term | Definition |
|---|---|
| Item-level request | A request placed against a specific item record |
| Title-level request | A request placed against an instance; FOLIO chooses the filling item |
| Page request | Request used when an item is currently available |
| Hold request | Request used when an item is not currently available |
| Recall request | Urgent request for a not-available item; may shorten the current loan due date |
| Pickup service point | Service point where the patron collects the item |
| Retrieval service point | Service point associated with retrieving the requested item; recent focus-release coverage adds UI, filter, export, and ECS checks around this field |
| Hold shelf expiration date | Date after which an awaiting-pickup request becomes pickup expired |
| Request expiration date | Optional date after which an open not-yet-filled request becomes unfilled |
| Hold shelf clearance report | Report listing expired or cancelled hold-shelf requests whose items need re-shelving |
| Batch requesting | Patron services workflow for creating multiple requests in one operation |
| Request level | `Item` or `Title` (`requestLevel` enum in mod-circulation `ramls/request.json`; filter labels `filters.requestLevel.item` / `.title`) |
| Fulfillment preference | `Hold Shelf` or `Delivery` (`fulfillmentPreference` enum, `ramls/request.json`; UI labels `requestMeta.fulfillment.holdShelf` = `Hold Shelf`, `requestMeta.fulfillment.delivery` = `Delivery`) |
| For use at location (UAL) | Reading-room loan: the loan policy that would apply to this requester + item has "For use at location" enabled; the item must not leave the location (Confluence 1779171329; CIRC-2457; UIREQ-1327) |
| Intermediate request | ECS/mediated-workflow request record that lives in the Central tenant on behalf of a Secure-tenant request (CIRC-2182; UIREQ-1254) |
| ECS TLR primary / secondary request | "Consortium request in the Consortium tenant" (ECS TLR primary request) vs "Consortium request in a local tenant" (ECS TLR secondary request) — distinguished by the `ecsRequestPhase` field (UIREQ-1088 developer notes) |
| Anonymized request | Closed request whose requester/proxy/delivery-address data was removed (CIRC-2292); UI shows `Anonymized` (`requestMeta.anonymized`) |

---

## Record Hierarchy / Architecture

```text
Instance
└── Holdings
    └── Item
        └── Request queue
            ├── Item-level requests
            └── Title-level requests associated back to the instance

Related records and services:
- User / requester record
- Pickup service point
- Retrieval service point
- Request policy and circulation rules
- Check in / Check out / mod-patron / title-level request services
```

---

## Lifecycle / Workflow

```text
Open - Not yet filled
  -> Open - In transit
  -> Open - Awaiting pickup
  -> Closed - Filled

Open - Not yet filled
  -> Open - Awaiting delivery
  -> Closed - Filled

Open - Not yet filled
  -> Closed - Cancelled
  -> Closed - Unfilled

Open - Awaiting pickup
  -> Closed - Filled
  -> Closed - Cancelled
  -> Closed - Pickup expired
```

| Status | Description |
|---|---|
| Open - Not yet filled | Request exists and has not yet been routed to the next fulfillment stage |
| Open - In transit | Item has been checked in and is moving to the pickup service point |
| Open - Awaiting pickup | Item is on hold shelf at the pickup service point |
| Open - Awaiting delivery | Delivery request was closed at check in without immediate checkout |
| Closed - Filled | Patron received the item through hold shelf checkout or close-and-check-out delivery flow |
| Closed - Cancelled | Request was cancelled by staff or patron |
| Closed - Pickup expired | Hold shelf expiration passed before pickup |
| Closed - Unfilled | Request expiration passed while still open and not yet filled |

Important exception: Open - Awaiting pickup is governed by the hold shelf expiration date, not the request expiration date.

The eight status strings above are the exact `status` enum of mod-circulation `ramls/request.json` and the `filters.requestStatus.*` labels in `en.json` — no other request status exists on master (UIREQ-956/957/958 "Open - Not on shelf" and UIREQ-584 "Reset to Open - Not yet filled" were closed **Won't Do**; do not write cases for them).

Anonymization transition (CIRC-2292): only `Closed - Filled`, `Closed - Cancelled`, `Closed - Pickup expired`, `Closed - Unfilled` requests can be anonymized; an anonymization attempt on any `Open - *` request makes no change. Anonymization does not change the status.

---

## Main Sections / UI Structure

### Panes

- Requests [from cases]
- Search & filter [from cases]
- Request details [from cases; confirmed `request.detail.title`]
- Request Detail [from cases, verify in env — the translation is `Request details`; prefer that]
- Notes [from cases]
- Request queue page: heading `Request queue on instance • {title} /{author}.` (`requestQueue.heading`), sections `Fulfillment in progress` (`requestQueue.fulfillmentInProgressSectionTitle`) and `Open - Not yet filled` (`requestQueue.notYetFilledSectionTitle`); empty texts `There are no requests in fulfillment.` / `No requests found`; `Refresh queue` button (en.json)
- New request form (`actions.newRequest` = `New request`): accordions `Item information` (`item.information`), `Title information` (`title.information`), `Requester information` (`requester.information`); `Create title level request` checkbox (`requests.createTitleLevelRequest`) expected only when "Allow title level requests" is enabled in Settings > Circulation > Title level requests [verify in env; UIREQ-1358 "checkbox is missing" was closed Cannot Reproduce]

### Accordions

- Staff notes [from cases]
- Requester information [from cases, verify in env]
- Loan and availability [from cases, verify in env]
- Requests [from cases, verify in env]

### Key Navigation Paths

- Requests > Search & filter [from cases]
- Requests > Actions > New [from source, verify in env]
- Requests > Request details > Actions > Move request [from source, verify in env]
- Requests > Request details > Actions > Reorder queue [from source, verify in env]
- Requests > Actions > `Export search results (CSV)` [en.json `exportSearchResultsCsv`; button id `exportToCsvPaneHeaderBtn` in fragment `requests.js`. docs.folio.org writes "Export search results to CSV" — the translation wins]
- Requests > Actions > `Hold shelf clearance report {currentServicePoint} (CSV)` [en.json `exportExpiredHoldShelfCsv`; button id `exportExpiredHoldsToCsvPaneHeaderBtn`; separate from the regular CSV export; disabled when the signed-in service point has no expired holds. Earlier versions of this file called it "Export expired holds to CSV" — that is only the fragment method name, not a UI label]
- Requests > Actions > `Print pick slips {sp}` / `Print selected pick slips {sp}` [en.json `printPickSlipsForSp`, `printSelectedPickSlipsForSp`; `{sp}` = signed-in service point]; row-level `Single print` (`requests.singlePrint`)
- Requests > Actions > `Print search slips {sp}` [en.json `printSearchSlipsForSp`; earlier text "Print hold request search slips" is not a UI label; option needs Settings > Circulation > Print hold requests > "Allow print hold requests (Open - Not yet filled)" (docs)]
- Loading texts while slips are fetched: `Pick slips are loading, please wait` (`pickSlipsLoading`), `Search slips are loading, please wait` (`searchSlipsLoading`), `Print options loading in progress. It might take a few seconds, please be patient.` (`printInProgress`)
- Request details > Actions > `Duplicate` (`actions.duplicateRequest`), `Move request` (`actions.moveRequest`), `Reorder queue` (`actions.reorderQueue`), `Edit` (`actions.edit`), `Cancel request` (`cancel.cancelRequest`)

---

## Exact UI Texts

> Use these verbatim in expected results only after validating the target tenant when a string is marked verify in env.

### Modal / Dialog Titles

| Modal | Trigger | Source |
|---|---|---|
| Awaiting pickup for a request | Check in flow when item reaches hold shelf | [from cases] |
| Confirm request cancellation | Cancel an open request | [from cases] |
| Select item | Item/title selection workflows | [from cases] |
| Select request type | Request type resolution when multiple types are possible | [from cases] |
| Route for delivery request | Check in flow for delivery requests | [from cases] |
| New request | New request workflow | [from cases, verify in env] |
| Request not allowed | Blocked request workflow | [from cases, verify in env] |
| Are you sure? | Generic confirmation dialog | [from cases] |

### Button Labels (key actions)

Actions, Save & close, Save, Edit, Search, Reset all, New, Confirm, Cancel, Continue, Print [from cases]

### New Request form — fields (confirmed C542, C545)

`Item barcode` field, `Requester barcode*` field, `Request Type*` dropdown (options: `Page`, `Hold`, `Recall` — which appear depends on item status), `Fulfillment preference` (Hold shelf / Delivery), `Pickup service point*` dropdown. Request types offered depend on item status: an Available item offers `Page`; a checked-out item offers `Hold` / `Recall`.

**Requester lookup works even without a barcode (confirmed, C10956):** a patron with no barcode on their record must still be selectable and submittable as the requester via the `Requester` lookup (name/other identifier search), not just via the barcode field — this was a past regression (`UIREQ-444`/`UIREQ-849`, "Unable to select requester without barcode" / "Cannot submit a request with a user without barcode"). Include a barcode-less-patron scenario when testing request creation, don't assume every requester has one.

### Empty-field display convention: literal `-`, never blank (confirmed C680499)

Any field with no underlying data value renders as a literal `-` (dash), never an empty cell — confirmed consistently across multiple, otherwise-unrelated surfaces in one pass: the `Select item` modal's `Barcode` column (both from New-request title-level-item-selection and from `Move request`), the `Reorder queue` page's `Item barcode` and `Requester barcode` columns, and the main Requests results table's `Year`, `Item barcode`, `Effective call number string`, `Requester barcode`, and `Proxy` columns. When constructing test data specifically to exercise "missing field" display, create the item/user with only required fields and assert `-` in each of these locations rather than assuming an empty string or blank cell.

### Result list columns (confirmed from `requests.js` fragment, 2026-08-06)

**Fixed visible columns:** `Request date`, `Title`, `Year`, `Type` (Request type), `Item barcode`, `Request status`, `Queue position`, `Requester`, `Requester barcode`, `Proxy`

**Optional toggle columns** (managed via Actions > Show columns; unchecked by default): `Pickup service point`, `Retrieval service point`, `# Copies`, `Printed`

- `# Copies` (`requests.copies`) — **number of times the pick slip for this request was printed** (not the number of item copies — the earlier wording here was wrong; UIREQ-1119/UIREQ-1121, docs "Showing the pick slip print history"); empty = never printed; NaN/empty cells sort last ascending
- `Printed` (`requests.printed`) — last printing user `lastName, firstName middleName` + date/time in the same locale format as `Request date` (UIREQ-1121); when that user was deleted only the date is shown (UIREQ-1305); empty cells sort last ascending
- `# Copies` and `Printed` exist only when Settings > Circulation > View print details > "Enable view print details (Pick slips)" is enabled (UIREQ-1121; docs)
- `Retrieval service point` (`requests.retrievalServicePoint`) — sortable with pagination and included in CSV (UIREQ-1188, UIREQ-1189, UIREQ-1191; Cypress C627236, C627454)
- In ECS with mod-tlr enabled the `Proxy` column is not offered (UIREQ-1185)
- Anonymized requests list `Requester` = `Anonymized`, `Requester barcode` = `-`, `Proxy` = `-` (UIREQ-1314); a deleted (not anonymized) requester shows `Unknown user` (`errors.user.unknown`) without a link (UIREQ-1312)
- `Title` with missing instance data (e.g., Inn-Reach transaction cancelled by the lender deletes instance/holdings/item) shows `-`, not clickable, and the app must not crash (UIREQ-1365, ui-requests 13.1.0)
- `Year` shows all years when the publication has several (UIREQ-1363, 13.0.2)

When testing column visibility or sorting, enable the relevant toggle column via Actions first, then assert the column header and sort behavior.

### Search & filter — filter accordions (confirmed C540, C541)

`Request type`, `Request status`, `Request level`, `Tags`, `Pickup service point`. Empty-result message: `No results found. Please check your filters.` `Reset all` clears all filters; filters persist across app navigation.

Master (`src/components/RequestsFilters/RequestsFilters.js`, `src/constants.js`) adds two more accordions: `Retrieval service point` (ECS and non-ECS since 12.0.2, UIREQ-1190/UIREQ-1294) and `Print status` (`requestMeta.printStatus`, options `Printed` / `Not printed`; only visible when "Enable view print details (Pick slips)" is on; selection persists across apps; CSV export must work with either option selected — UIREQ-1119). Filter option labels: Request type `Holds` / `Pages` / `Recalls` (`filters.requestType.*` — plural, unlike the form's `Hold`/`Page`/`Recall`); Request level `Item` / `Title`; Request status = the 8 status strings.

**Search box fields** (`src/utils.js` `requestSearchFields`): request `id` (exact), `requesterId`, requester barcode (starts-with, `==`, UIREQ-640), instance title (keyword), `instanceId`, item barcode (starts-with), `itemId`, `itemIsbn` (normalized ISBN — hyphenated and unhyphenated forms both match, UIREQ-1355 / C343043), call number and full call number (starts-with). Searching by the value printed from the `{{request.requestID}}` / `{{request.barcodeImage}}` slip token finds the request (UIREQ-1232). docs.folio.org describes only title / item barcode / requester barcode / call number — the source list is wider. Karate: `requests-extended.feature` "Search requests by item barcode, title, request id, and requester barcode fragment"; `requests.feature` "Search requests of all types and verify request UUID appears in search results". Double quotes in the search term are supported (Cypress C380518).

### Staff-slip token editing (confirmed C543, C544)

`Add token` modal (opened via the `{ }` icon in the Template content section); tokens land in the `Display` text field. Token syntax is `{{token.name}}`, e.g. `{{requester.firstName}}`, `{{requester.lastName}}`, `{{item.title}}`, `{{item.barcode}}`, `{{item.effectiveLocationSpecific}}`.

### Toast messages (confirmed — `ui-requests/en.json` master 2026-11-24; `en_US.json` identical)

| Event | Toast text | Key |
|---|---|---|
| Request created | `Request has been successfully created for {requester}` | `createRequest.success` |
| Request create failed | `This request was not placed successfully` | `createRequest.fail` |
| Request edited | `Request has been successfully edited for {requester}` | `editRequest.success` |
| Request edit failed | `This request was not saved successfully` | `editRequest.fail` |
| Request duplicated | `Request has been successfully created for {requester}` | `duplicateRequest.success` |
| Request duplicate failed | `This request was not placed successfully` | `duplicateRequest.fail` |
| Request moved | `Request has been moved successfully` | `moveRequest.success` |
| Queue reordered | `Queue was successfully updated` | `requestQueue.reorderSuccess` |
| Queue changed by someone else | modal `Request queue is out of sync` / `This request queue has been updated by another person or process and may be out of synch.` | `errors.sync.requestQueueLabel` / `.requestQueueBody` |
| Reorder failed | modal `Request queue error` / `Request queue reorder failed` | `errors.unknown.requestQueueLabel` / `.requestQueueBody` |

### Request-not-allowed / validation errors (negative-case gold — confirmed)

| Condition | Message text |
|---|---|
| Only checked-out can be recalled | `Only checked out items can be recalled` |
| Only checked-out can be held | `Only checked out items can be held` |
| Hold not allowed for patron+title | `Hold requests are not allowed for this patron and title combination` |
| No request type available (title) | `None available for this title and patron combination` |
| No request type available (item) | `None available for this item and patron combination` — appears **inline under the `Request type` dropdown**, not as a toast; confirmed triggers include item statuses `Withdrawn`, `Declared lost`, `Claimed returned`, and `Lost and paid` (verify each individually — C9193, C10931) |
| Requester already has item on loan | `This requester already has this item on loan` |
| Requester already has loan for one of instance's items | `This requester already has a loan for one of the instance's items` |
| Requester already has open request for item | `This requester already has an open request for this item` |
| Requester already has open request for instance | `This requester already has an open request for this instance` |
| SP is not a pickup location | `Service point is not a Pickup location` |
| Hold-shelf needs pickup SP | `Hold shelf fulfillment requests require a Pickup service point` |
| Request already closed | `Error: The Request has already been closed.` |
| User cannot be proxy for self | `User cannot be proxy for themselves` |
| Item barcode not found | `Item with this barcode does not exist` |
| User barcode not found | `User with this barcode does not exist` |
| Instance HRID/UUID not found | `Instance with this HRID or UUID does not exist` |
| Requester is Inactive | Toast `This request was not placed successfully`; modal `Request not allowed` body `Inactive users cannot make requests.`, `Close` button (confirmed: both `Close` and the `X` icon dismiss it; neither creates the request) — applies to Page, Hold, and Recall alike (C1385311/C1385312, item- and title-level; Cypress `item-request-cannot-be-created-for-inactive-patron.cy.js`, `title-request-cannot-be-created-for-inactive-patron.cy.js`). `Request not allowed` = `requestNotAllowed`, `Close` = `close` in ui-requests en.json; the body text is not in ui-requests en.json (comes from the backend error) |
| Pickup SP no longer allowed | `One or more Pickup locations are no longer available` (`errors.requestPickupServicePointIsNotAllowed`) |
| Requester already has open request for one of instance's items | `This requester already has an open request for one of the instance's items` (`errors.itemOfThisInstanceAlreadyRequested`) |
| Page TLR without item | `Cannot create a title level page request for this instance ID with no item ID` (`errors.cannotCreatePageTlrWithoutItemId`) |
| Hold/Recall TLR while a pageable item is available | `Hold/Recall title level request not allowed: pageable available item found for instance` (`errors.holdAndRecallTlrNotAllowedPageableAvailableItemFound`) |
| Move TLR page to same item | `Not allowed to move title level page request to the same item` (`errors.movingRequestToTheSameItem`) |
| Invalid request level (API-originated) | `Request level must be one of the following: "Item", "Title"` (`errors.requestLevelIsNotAllowed`) |
| Invalid fulfillment preference | `Fulfillment preference must be one of the following: Hold Shelf, Delivery` (`errors.fulfillmentPreferenceIsNotAllowed`) |
| Form validation | `Please select an item` / `Please select an item and hit enter` / `Please select an instance and hit enter` / `Please fill out requestor information to continue` / `Please select a request type` (`errors.selectItem`, `.selectItemRequired`, `.selectInstanceRequired`, `.selectUser`, `.requestType.selectItem`) |

### Cancel-request modal (confirmed, incl. field-gating detail from C3533)

`Confirm request cancellation` heading; body `<b>{title}</b> will be <b>cancelled</b>`; fields `Reason for cancellation`, `Notify patron`, `Additional information for patron` (key `cancel.additionalInfoLabel` = `Additional information for patron {required}`; placeholders `Enter more information about the cancellation (required)` / `Enter more information about the cancellation (optional)` — `cancel.additionalInfoPlaceholderRequired` / `...Optional`). UIREQ-777 (configurable "required" flag per cancellation reason) and UIREQ-470 (staff note on cancel) were closed **Won't Do** — only `Other` makes the text required. Blocked-request modal: heading `Patron blocked from requesting`, `Reason for block`, `View block details`, `Request not allowed`.

**Patron-block override on request creation — confirmed details (C170385):** the modal shows at most **3 blocks** even if the patron has more. Closing the modal (without overriding) does NOT let the request save — clicking `Save & close` again re-triggers the same block modal; only `Override` clears the gate. **There is no comment field on this override** (unlike Check-out/Renewal override, which requires a comment) — the team's own case notes this as a deliberate current asymmetry ("future development planned to add a comment during override"). Don't write a case that expects a required-comment step here; that belongs to check-out.md/loans.md's override patterns, not this one.

**Field-gating logic (confirmed, C3533):** `Reason for cancellation` options come from `Settings > Circulation > Request cancellation reasons`. Once **any** reason is selected, the `Additional information for patron` free-text field appears — but it's only **required** when the reason is `Other`; for every other reason it's optional. `Confirm` stays disabled only when reason = `Other` and the text field is empty; for any other reason `Confirm` is enabled as soon as a reason is picked. `Back` closes the modal without cancelling. Don't write a case that requires text for every reason, or that never shows the field for non-Other reasons — both are wrong.

### Patron comments on a request (confirmed, C199704–C199708 family — previously undocumented)

`Patron comments` is a free-text field on the `New request` form, fillable at creation time only — once the request is saved, re-opening it via `Actions > Edit` shows the same field but **uneditable** (write-once from the UI). Real coverage of this feature spans five distinct surfaces the team tests separately — treat each as its own scenario, not one combined case:
1. **Creation**: field appears on `New request`, accepts any text, persists on save (C199704).
2. **CSV export**: patron comment column is included in the Requests CSV export (C199705).
3. **Hold shelf clearance report**: patron comment appears in that report's row for the request (C199706).
4. **Staff slip token**: the token is `request.patronComments` (Settings > Circulation > Staff slips > token picker `{}`) and must be configured and verified separately in **all three** applicable templates — `Hold`, `Pick Slip`, and `Request Delivery` — each triggered by a different check-in flow (Hold: "Awaiting pickup for a request" modal → print slip; Pick Slip: Requests app "Print pick slips for `<service point>`"; Request Delivery: "Route for delivery request" modal → print slip). Don't test only one slip type and assume the others work the same way (C199707).
5. **Check-in "Awaiting pickup for a request" modal**: when checking in the item, the modal shown at that point displays the patron comment associated with the request (C199708) — this is a cross-app effect landing in Check In, verify it there per this skill's "verify effects at their real destination" rule.

### Move request / reorder queue (confirmed)

`Move request` action opens the `Select item` modal, which must list **every requestable/available item on the Instance** — this was a past regression when an Instance has **more than 10 holdings**: pagination/limit bugs caused some requestable items to be silently missing from the list (`UIREQ-929`). When testing Move Request on a multi-holdings Instance, use 10+ holdings and assert the full item set appears, not just that the modal opens (C388490). `Select request type`; second-position message `Requests cannot be moved above page requests, even when fulfillment has not begun. This request will be moved to position 2 in the request queue.` Reorder confirm: `Move to position 2` / `Requests cannot be displaced from position 1 when fulfillment has begun.`

Exact keys: Move-request second-position modal title `Request moved to position 2 of the queue` (`moveRequest.secondPositionTitle`) + `moveRequest.secondPositionMessage`; type picker `Select request type` / `Please select allowed type` / `Confirm`. Reorder-queue confirm modal: title `Request moved to position 2 of the queue`, buttons `Move to position 2` / `Leave in current position` (`requestQueue.confirmReorder.confirm` / `.cancel`), messages `requestQueue.confirmReorder.message1` and `message2` = `Requests cannot be moved above page requests, even when fulfillment hasn't begun.` Queue-page columns: `Order`, `Status`, `Item barcode`, `Enumeration`, `Chronology`, `Volume`, `Patron comments`, `Request status`, `Pickup/Delivery`, `Requester`, `Requester barcode`, `Patron group`, `Request date`, `Type`, `Request expiration`, `Hold shelf expiration` (`requestQueue.*`). UIREQ-301 (move above a page) and UIREQ-577 (Move for `Open - In transit`) were closed Won't Do — current behaviour stands. A moved request goes to the end of the target item's queue (Cypress C6652).

> `No results found. Please check your filters.` (empty filter results) is a shared stripes component string, confirmed from cases C540/C541.

### For use at location (UAL / reading-room) on requests (Trillium R1 2026, ui-requests 13.0.0)

- Feature is off unless mod-circulation is deployed with `ENABLE_FOR_USE_AT_LOCATION=true` (launchDescriptor default `false`; CIRC-2457). On a tenant without it, no UAL label appears — state the env prerequisite in Preconditions.
- UAL is a **loan policy** setting (Settings > Circulation > Loan policies > "For use at location" + hold shelf expiration period in days/weeks/months); circulation rules map loan type / material type / patron group to that policy (Confluence 1779171329).
- New request form: `Item information` shows `Loan type` (`loanType`, read-only from the item; UIREQ-1282); `Request information` shows `For use at location` (`forUseAtLocation`) with a green check icon **when the loan policy that a loan for this requester + item would get is UAL** — it needs both item and requester and works for Available items (no current loan) (UIREQ-1327 replaced the first, current-loan-based implementation of UIREQ-1282; UIREQ-1326). Confluence writes the label as "Use at location"; the translation is `For use at location`.
- Request details page shows `Loan type` for Page and Hold requests (UIREQ-1328; backend adds `item.loanTypeId` / `item.loanTypeName` to `circulation/requests` responses, CIRC-2441).
- When the request will be UAL, `Delivery` is **not** offered as fulfillment preference even if the requester's preference is Delivery — only Hold Shelf (UIREQ-1329).
- Downstream (Check in / Check out, not this app): check-in action `Keep on hold shelf` / `Close loan and return item` / `Ask for action`, endpoints `POST /circulation/hold-by-barcode-for-use-at-location` and `/circulation/pickup-by-barcode-for-use-at-location` (Confluence 1779171329; ModuleDescriptor). Verify those effects in check-in.md / check-out.md.

### Anonymized, deleted, and erased data display

| Situation | Request details | Results list | CSV | Source |
|---|---|---|---|---|
| Request anonymized | Requester name `Anonymized` (`requestMeta.anonymized`), no link; barcode `-`; Proxy box not shown; delivery address gone | `Requester` = `Anonymized`, `Requester barcode` = `-`, `Proxy` = `-` | requester name / barcode / delivery address exported empty | UIREQ-1313, UIREQ-1314 (+ comments), CIRC-2449 |
| Requester user deleted (not anonymized) | `Unknown user` (`errors.user.unknown`), no link; barcode `-` | same | — | UIREQ-1312 |
| User who last printed the slip deleted | — | `Printed` shows only the date | export still generated, date only | UIREQ-1305; Karate `print-events-extended.feature` |
| Instance/item erased (Inn-Reach cancel by lender) | page opens, no crash | `Title` = `-`, not a link | — | UIREQ-1365 (C1395025) |
| Note assigned to an anonymized request | `Requester: Anonymized` under Assigned | — | — | UIREQ-1313 comment |

Anonymization itself is **backend-only** on master: `POST /request-anonymization/{requestId}` (CIRC-2364) plus scheduled processing; there is no UI action yet (UIREQ-1359 "FE - Procedural Capability Set - Manual Request Anonymization - Single" is open). Out of scope per Confluence 1056210996: UI tools for starting anonymization.

---

## Status Values

### Request status

Open - Not yet filled, Open - In transit, Open - Awaiting pickup, Open - Awaiting delivery, Closed - Filled, Closed - Cancelled, Closed - Pickup expired, Closed - Unfilled

### Request type

Page, Hold, Recall

### Request level / fulfillment preference

`Item`, `Title` · `Hold Shelf`, `Delivery` (mod-circulation `ramls/request.json` enums). Required fields on `POST /circulation/requests`: `requesterId`, `requestType`, `requestDate`, `fulfillmentPreference` (`required` in `ramls/request.json`); `instanceId`, `holdingsRecordId`, `requestLevel` are not mandatory for backward compatibility (Karate `requests.feature` "VuFind integration - backward compatibility").

### Item statuses that appear in the Item information accordion (`item.status.*` in en.json)

Aged to lost, Available, Awaiting delivery, Awaiting pickup, Checked out, Claimed returned, Declared lost, In process, In process (non-requestable), In transit, Intellectual item, Long missing, Lost and paid, Missing, On order, Paged, Unavailable, Unknown, Withdrawn, Recently returned, Available in ASR, Retrieving from ASR, Missing from ASR, Order closed, Restricted. (Custom item statuses — UIREQ-475/480 — were closed Won't Do.)

---

## API Surface (mod-circulation master, `descriptors/ModuleDescriptor-template.json`)

| Method | Path | permissionsRequired |
|---|---|---|
| GET / POST / DELETE | `/circulation/requests` | `circulation.requests.collection.get` / `circulation.requests.item.post` / `circulation.requests.collection.delete` |
| GET / PUT / DELETE | `/circulation/requests/{id}` | `circulation.requests.item.get` / `.item.put` / `.item.delete` |
| POST | `/circulation/requests/{id}/move` | `circulation.requests.item.move.post` |
| GET | `/circulation/requests/queue/item/{id}` | `circulation.requests.queue-item.collection.get` |
| POST | `/circulation/requests/queue/item/{id}/reorder` | `circulation.requests.queue.item-reorder.collection.post` |
| GET | `/circulation/requests/queue/instance/{id}` | `circulation.requests.queue-instance.collection.get` |
| POST | `/circulation/requests/queue/instance/{id}/reorder` | `circulation.requests.queue.instance-reorder.collection.post` |
| POST | `/circulation/requests/instances` (title-level create) | `circulation.requests.instances.item.post` |
| GET | `/circulation/requests/allowed-service-points` | `circulation.requests.allowed-service-points.get` |
| GET | `/circulation/requests-reports/hold-shelf-clearance/{id}` | `circulation.requests.hold-shelf-clearance-report.get` |
| POST | `/request-anonymization/{requestId}` (closed requests only) | `circulation.requests.anonymize.item.post` |
| POST | `/circulation/print-events-entry` (pick-slip print log) | `circulation.print-events-entry.item.post` |
| GET | `/circulation/rules/request-policy`, `/circulation/rules/request-policy-all` | `circulation.rules.request-policy.get`, `.request-policy-all.get` |
| POST | `/circulation/hold-by-barcode-for-use-at-location`, `/circulation/pickup-by-barcode-for-use-at-location` | `circulation.hold-by-barcode-for-use-at-location.post`, `circulation.pickup-by-barcode-for-use-at-location.post` |
| timers | `/circulation/request-scheduled-notices-processing`, `/circulation/scheduled-request-anonymize-processing` | — |

The UI also calls mod-circulation-bff: `circulation-bff/requests/search-instances`, `circulation-bff/requests/allowed-service-points?operation=…&requestId=…`, `circulation-bff` pick-slips / search-slips (UIREQ-1154; `src/routes/RequestsRoute.js`; package.json subPermissions `circulation-bff.*`). Request-creation errors return **422** with an error schema, not 500 (CIRC-2104; CIRC-1944/CIRC-1964 re-tested as 422).

## Settings That Affect Requests

| Setting | Effect | Source |
|---|---|---|
| Settings > Circulation > Title level requests: "Allow title level requests" (+ ECS "Enable consortium title level requests (TLR)", "Exclude tenants") | TLR checkbox / title look-up; cannot be disabled while open TLRs exist | docs; CIRC-2697 preconditions; Karate `tlr-requests.feature` "TLR Page request fails when TLR feature is disabled and succeeds when it is enabled" |
| Settings > Circulation > View print details: "Enable view print details (Pick slips)" | `# Copies`, `Printed` columns and `Print status` filter | UIREQ-1119, UIREQ-1121; docs |
| Settings > Circulation > Print hold requests: "Allow print hold requests (Open - Not yet filled)" | `Print search slips {sp}` | docs |
| Settings > Circulation > Request cancellation reasons | `Reason for cancellation` options | C3533 |
| Settings > Circulation > Staff slips (Pick slip, Search slip (Hold requests), Hold, Request delivery, Transit) | print templates and tokens | C199707; Karate `requests-extended.feature` (staffUsername, requester.departments) |
| Settings > Tenant > Service points: Pickup location, Hold shelf expiration period | pickup SP list; hold-shelf clearance; an SP that stops being a pickup location is removed from request-policy allowed SPs | docs; Karate `requests-extended.feature` |
| Request policy: allowed request types, "Allow some pickup service points" | type options and pickup SP list (Delivery requests must still save when SPs are restricted — CIRC-2122) | Karate `requests.feature`, `tlr-requests.feature` |
| Patron group limits (max items charged out, overdue recalls, fee/fine balance, lost items, overdue items, recall overdue days) | block request creation | Karate `requests.feature` (mod-patron-blocks conditions) |
| Loan policy "For use at location" + env `ENABLE_FOR_USE_AT_LOCATION` | UAL label, no Delivery | CIRC-2457; UIREQ-1327/1329 |
| Request anonymization settings (circulation settings; `circulation.request-anonymization-settings.get/put`) | automatic anonymization timing | CIRCSTORE-629; ModuleDescriptor permissionSets; Confluence 1056210996 |
| mod-circulation env `ECS_TLR_FEATURE_ENABLED` (default `false`) | ECS TLR via mod-tlr | ModuleDescriptor launchDescriptor |

---

## Capability Sets (Eureka)

Curated capability set list for Requests authoring:

| Action | Capability Set |
|---|---|
| View requests | `Data - UI-Requests - View` |
| View and create requests | `Data - UI-Requests - Create` |
| View, edit, and cancel requests | `Data - UI-Requests - Edit` |
| All request functions | `Data - UI-Requests - Manage` |
| Move request to another item | `Procedural - UI-Requests MoveRequest - Execute` |
| Reorder queue | `Procedural - UI-Requests ReorderQueue - Execute` |
| Title look-up while creating title-level requests | `Data - UI-Inventory Instance - View` |

Observed older cases also reference circulation settings permissions around print-hold-requests and search-slip settings. Where those settings are part of the scenario, verify the current Eureka capability-set names in tenant before using them verbatim.

Confirmed mapping (docs.folio.org Requests > Permissions table + ui-requests `package.json` permissionSets, master 13.1.0):

| Okapi permission (display name) | Eureka resource / type / action | Key subPermissions / notes |
|---|---|---|
| `ui-requests.view` — Requests: View | UI-Requests / Data / View | search & view, hold-shelf clearance report, pick/search slips (`circulation-bff.pick-slips/search-slips`), `circulation.print-events-entry.item.post` → **printing slips works with View alone** (UIREQ-1217, 12.0.0) |
| `ui-requests.create` — Requests: View, create | UI-Requests / Data / Create | + `circulation.requests.item.post`, `circulation-bff.requests.post`, allowed-service-points, patron blocks, `search.instances.collection.get` + inventory reference data → **title look-up works with View, create** (UIREQ-1296) |
| `ui-requests.edit` — Requests: View, edit, cancel | UI-Requests / Data / Edit | + `circulation.requests.item.put`, `tags.item.post` |
| `ui-requests.all` — Requests: All permissions | UI-Requests / Data / Manage | view + create + edit + move + reorder |
| `ui-requests.moveRequest.execute` — Requests: Move to new item, reorder queue | UI-Requests MoveRequest / Procedural / Execute | includes `ui-requests.reorderQueue.execute` |
| `ui-requests.reorderQueue.execute` — Requests: Reorder queue | UI-Requests ReorderQueue / Procedural / Execute | queue endpoints for item + instance |
| `circulation.requests.anonymize.item.post` — Requests: anonymize single closed request | backend capability "UI-Requests Anonymize-Single-Request" / Procedural / Execute (CIRC-2434) | API only; UI set UIREQ-1359 still open — verify name in tenant |

Cypress permission specs: `permissions-requests-view-create.cy.js` (C3485), C3483 (View), C3484 (View, edit, cancel).

---

## Common Verification Patterns

### Request status transition verification

```text
Action:   Complete the request workflow step that moves the request forward.
Expected: Request details show the exact updated Request status value and the item status reflects the same fulfillment stage.
```

### Retrieval service point list-column verification

```text
Action:   Add the Retrieval service point column or apply the Retrieval service point filter in the Requests result list.
Expected: The result list displays the Retrieval service point value correctly for item- and title-level requests, and filtering/sorting/export use the same value consistently.
```

### Request export verification

```text
Action:   Run Actions > `Export search results (CSV)` from a filtered Requests result set.
Expected: The generated CSV contains the visible request data plus additional request/item fields, including Retrieval service point when that column is part of the tested release scope.
```

### Delivery-route verification at check in

```text
Action:   Check in an item with Delivery fulfillment and choose one of the Route for delivery request dialog options.
Expected: Close and check out changes the request to Closed - Filled; Close changes the request to Open - Awaiting delivery.
```

### Batch / multi-item requesting — API (confirmed C1292036, C1301731; refs MODPATRON-253, MODREQMED-161)

Batch (a.k.a. multi-item) requesting is an **edge-patron API flow**, not a UI screen — cases are Postman-style: method / URL / headers / body, and the expected result is the response code + body fields. Steps and preconditions are written as bulleted API blocks.

- **Create a batch request:** `POST {{host}}/patron/account/{userId}/instance/{instanceId}/batch-request`, header `X-Okapi-Tenant = {tenantId}`, body:
  ```json
  { "patronComments": "...", "requests": [ { "itemId": "...", "pickUpLocationId": "..." }, { "itemId": "...", "pickUpLocationId": "..." } ] }
  ```
  A successful response returns each request line; in a Secure tenant each line carries `"mediatedRequestStatus": "Pending"` (batch requests placed in the Secure tenant are mediated). Retrieve a patron's batch requests via `GET {{host}}/patron/account/{userId}` with the batch include parameter.
- **Max items per batch is a configurable setting**, per Member tenant, via `POST /settings/entries` (create, 201) / `PUT /settings/entries/{settingId}` (update, 204): `{ "scope": "mod-requests-mediated.manage", "key": "multiItemBatchRequestItemsValidation", "value": { "maxAllowedItemsCount": N } }`. A batch exceeding `maxAllowedItemsCount` is rejected; raising the value then re-sending the same batch succeeds — so a limit case sets the value low, exceeds it, then raises it and retries.
- **Preconditions pattern for these ECS batch cases:** `Allow title level requests` checked in Central + Member + Secure, `Enable consortium title level requests (TLR)` checked in Central with no tenants in `Exclude tenants`; a Shared instance created in Central with a Holding in a Member tenant; ≥2 items whose material/loan types are shared. ECS variants repeat the call swapping the `userId` in the path and the `X-Okapi-Tenant` header for Member / Secure tenants.
- Related: `Create Mediated batch request for Secure patron via edge-patron` (C1250456) and `Create non-ECS batch requests via edge-patron` (C1322897) are the mediated and non-ECS variants of the same POST flow.

---

## ECS / Multi-Tenant Notes

Requests subtree section 96 includes ECS-specific focus-release coverage. Recent Sunflower and Trillium cases cover:

- Retrieval service point column behavior for local, ECS, and mediated item/title-level requests [from cases]
- Batch ECS request creation and mediated secure-patron batch request creation via edge-patron [from cases]

When writing ECS requests tests, specify:

- Central tenant versus member tenant ownership of the requester and item
- Starting affiliation and any affiliation switch step
- Whether the flow is local, ECS, mediated, or secure-patron mediated

ECS behaviour confirmed from Jira (ECS with mod-tlr enabled unless stated):

- `Duplicate` and `Move request` actions are hidden on Request details; if no action remains, the `Actions` button is hidden (UIREQ-1127, UIREQ-1128).
- Proxy is hidden: no "Who are you acting as?" modal on New request, no `Proxy` column option. Single-tenant proxy behaviour unchanged (UIREQ-1185).
- Open requests are edited only from the tenant where they were created — local request from its local tenant, ECS TLR primary request from the Consortium (Central) tenant (UIREQ-1088).
- Intermediate requests in Central (from mediated/Secure-tenant flow): Edit / Cancel / Move / Duplicate hidden; only `Reorder queue` remains, if the user has the permission (UIREQ-1254).
- Central-tenant Request details: item barcode in Item information is **not** a hyperlink (UIREQ-1306, ui-requests 12.0.3/11.0.9; story UIREQ-1307 to restore it is open).
- Retrieval service point is kept on the request after the item is checked in at a non-pickup SP and goes in transit (CIRC-2535) — check local, ECS ILR/TLR, and mediated variants (not Hold TLRs).
- ECS pick slips print at the SP shown in `Retrieval service point` (CIRC-2538; TLR variant CIRC-2399). Retrieval SP filter in Central lists the correct SPs (CIRC-2239).
- Batch requests for the same instance get unique, non-duplicated queue positions in Central and Data tenant (CIRC-2697, backport CIRC-2700 for Trillium).
- Hold shelf expiration notice for an ECS request is sent once, not twice (CIRC-2664, not reproduced on snapshot after fix).
- CSV export works for Hold TLR in ECS (UIREQ-1292).
- Search by normalized ISBN finds requests created after Mediated request confirmation in Central, Data, and Secure tenants (UIREQ-1355 test comment).
- Coverage: Cypress `cypress/e2e/consortia/requests/` (C624255–C624260 create ECS ILR/TLR Page/Hold/Recall; C624274–C624279 cancel); Karate `folio-ecs-circulation/.../ecs-requests/features/ecs-requests.feature` (create ILR/TLR ECS request via mod-circulation-bff), `mediated-requests/mediated-requests.feature`, `staff-slips/features/staff-slips.feature`; `edge-patron/.../consortia/features/batch-ecs-requests.feature`, `ecs-requests.feature`.

---

## Release-Focused Signals

Focus-release coverage in section 96 is concentrated in these areas:

1. Retrieval service point UI, filtering, sorting, export, and ECS visibility checks in Ramsons and Sunflower.
2. Requests CSV export edge cases and deleted-requester behavior in Trillium.
3. Batch requesting, patron settings, maximum item limits, and edge-patron API flows in Trillium.
4. Same-patron requests for different items on the same instance, request-policy editing, and request search by normalized ISBN in Ramsons.

5. Trillium R1 2026 (ui-requests 13.0.x, mod-circulation 24.5.x): For use at location label / loan type / no Delivery (UIREQ-1282, 1326, 1327, 1328, 1329; CIRC-2441, CIRC-2457); anonymized and deleted users display (UIREQ-1312, 1313, 1314; CIRC-2449); request anonymization API (CIRC-2292, 2364, 2434); ISBN search fix (UIREQ-1355); CSV with deleted printing user (UIREQ-1305); TLR with a Restricted item (CIRC-2439); pick slips sorted by location then shelving order (CIRC-1889).
6. Umbrellaleaf R2 2026 (ui-requests 13.1.0 in progress): erased-data dash link (UIREQ-1365); batch queue positions (CIRC-2697); ECS pick-slip SP (CIRC-2538); duplicate hold-shelf-expiration notice (CIRC-2664). No new ui-requests features beyond these on master as of 2026-11-24.

This means new manual cases for Requests should give extra attention to retrieval service point behavior, batch requesting, patron-services API coverage, ECS-mediated variants, UAL, and anonymized/deleted-user display before adding more legacy core-flow coverage.

---

## Existing Automated Coverage (check before writing a duplicate)

**Karate** (`folio-integration-tests`):
- `mod-circulation/src/main/resources/vega/mod-circulation/features/requests.feature` (36 scenarios; runner `ModCirculationTests.java`): page/hold/recall vs request policy allow/disallow and item statuses; patron note; cancel; move to another item on the same instance; pick slips for an SP; reorder item queue; hold shelf clearance report; VuFind backward compatibility; 6 patron-group-limit blocks; filter by call number; sort by SP name / shelving order; allowed service points for item and instance; pickup SP not allowed by policy; Hindi locale page request; request UUID in search.
- `.../features/requests-extended.feature` (runner `ModCirculationExtendedTests.java`): search by item barcode / title / request id / requester-barcode fragment; deleted or non-pickup SP removed from policy allowed SPs; filter by request level; `requester.departments` in pick slip; `staffUsername` token in Search slip, Request delivery, Transit slips.
- `.../features/tlr-requests.feature`: create / cancel TLR; reorder instance queue; mixed item+title queue order; TLR recall vs checkout of another item; TLR hold vs circulation rules / request policy; TLR hold with no items; pickup SP not allowed; TLR feature disabled → page fails.
- `.../features/print-events-extended.feature`: CSV export when the last printing user was deleted.
- ECS / edge-patron: see ECS notes above.

**Cypress** (`stripes-testing`): fragments `cypress/support/fragments/requests/` (`requests.js`, `newRequest.js`, `requestDetail.js`, `edit-request.js`, `move-request.js`, `requestsSearchResultsPane.js`, `selectInstanceModal.js`, `selectUser.js`); specs `cypress/e2e/requests/` (48 specs — e.g. C540/C541 type filters, C545–C547 create, C553/C554 requester look-up, C556 edit, C557 status workflow, C3533 cancel, C2379 sorting, C2497 hold shelf clearance, C15178 pickup SP filter, C9320/C374173 tags, C199704/C199705/C199708 patron comments, C343043 ISBN search, C553064 `# Copies`/`Printed` sort, C627236 retrieval SP column, C627454 retrieval SP sort with pagination, C1385311/C1385312 inactive patron, C745/C10956 request without barcode, C6652 move to end of queue), `cypress/e2e/title-level-request/` (49 specs — C350xxx/C353xxx TLR create/edit/duplicate/move/cancel/queue, C375939–C375949 TLR recall by item status, C411787 TLR hold with no items, C410741 pickup SP changed), `cypress/e2e/consortia/requests/` (12 specs). No Cypress or Karate coverage found for UAL on requests, anonymized display, or the request-anonymization API (grep of both repos, 2026-11-24).

---

## Key Business Rules for Test Cases

1. Request type options depend on current item status and circulation rules; not every item can offer Hold, Page, and Recall at once. (docs + cases)
2. Page requests are for available items, Hold requests are for unavailable items, and Recall requests are urgent requests that may shorten the current borrower's due date. (docs)
3. Request expiration date is optional; if it is set and the request remains Open - Not yet filled past that date, the request becomes Closed - Unfilled. (docs)
4. Open - Awaiting pickup ignores the request expiration date and closes only when the hold shelf expiration date passes. (docs)
5. Only open requests can be edited or cancelled; closed requests do not allow those actions. (docs)
6. Cancelling a Page request with no remaining queue returns the item to Available. (docs)
7. Cancelling a request while it is In transit or Awaiting pickup closes the request, but the item status does not change until the item is checked in. (docs)
8. A duplicate request cannot be saved for the same requester and the same item; at least one of those values must change. Recent Ramsons coverage confirms the allowed variant where the same patron requests different items on the same instance. (docs + cases)
9. Moving a request to a loaned target item on the same instance can trigger recall behavior and due-date changes for the current borrower. (docs)
10. Title-level requests require newly added items to be checked in before they can satisfy an existing open title-level request. (docs)
11. Title-level requesting cannot be disabled while open title-level requests exist. (docs)
12. Title-level requests are not supported for multi-volume sets; item-level requests must be used instead. (docs)
13. Delivery fulfillment requires delivery to be enabled on the patron record and a valid delivery address to exist. (docs)
14. Hold shelf clearance report must include cancelled-awaiting-pickup and pickup-expired items that need to be re-shelved. (docs + cases)
15. No request can be moved above a Page request in the queue (even when fulfillment has not begun), and a request whose fulfillment has begun cannot be displaced from position 1 — the UI offers `Move to position 2` / `Leave in current position`. (en.json `requestQueue.confirmReorder.*`, `moveRequest.secondPositionMessage` — corrected from "Page requests cannot be moved above other Page requests")
16. Reorder queue and Move request are separate capabilities and must be tested independently. (docs)
17. Title look-up for title-level request creation requires inventory instance view capability. (docs)
18. Pickup notices are sent at the end of the check-in session rather than immediately when the item reaches the hold shelf. (docs)
19. Cancellation reason Other requires additional information; for other reasons it is optional. (docs; C3533; en.json `cancel.additionalInfoPlaceholderRequired`/`...Optional`)
20. Retrieval service point is now a meaningful verification surface across list columns, filters, sorting, CSV export, printing, and ECS visibility. (cases)
21. Requests export must still succeed when prior printed-request data or deleted-requester data creates edge conditions. (cases)
22. Batch requesting obeys patron settings and multi-item request limits, including ECS and non-ECS edge-patron flows. (cases)
23. Patron-services API flows exist for item-level hold creation, title-level hold creation, cancellation, allowed-service-points resolution, and related request workflows; use API-focused manual cases only when the story explicitly targets those endpoints. (cases)
24. **A `Closed - Cancelled` (or any Closed-*) request has no Edit control anywhere in the UI, but the view still shows every field fully populated** — verify both: the absent edit affordance AND that the closed record's data is still intact/readable, not just one or the other. (cases — C3533)
25. Cancelling a request while `Awaiting pickup` doesn't resolve the item's status immediately — the item stays as-is until the next **check-in**, at which point it resolves per normal queue logic (`Available` if no other requests and the check-in service point matches the item's effective location; `In transit` if it must route elsewhere for reshelving or the next request; `Awaiting pickup` if another request needs the hold shelf). A cancel-while-awaiting-pickup case should include this follow-up check-in step and assert the resulting status, not stop at the cancel toast. (cases — C3533)
26. **An Inactive patron cannot have any new request placed for them** — Page, Hold, and Recall are all blocked identically (`Request not allowed` / `Inactive users cannot make requests.`), for both item- and title-level requests; verify no request record is created after the blocked attempt. (cases — C1385311, C1385312)
27. **Certain item statuses block new request creation outright** (see "No request type available (item)" in the errors table above) — `Withdrawn`, `Declared lost`, `Claimed returned`, and `Lost and paid` all produce the inline `None available for this item and patron combination` message under `Request type` rather than letting the request form proceed; verify each status individually rather than assuming they behave the same as `Checked out`/`Available`. (cases — C9193, C10931)
28. **A recall's due-date truncation and "Item recalled" notice fire only once per loan, on the *first* recall.** When a checked-out item with **zero** existing recalls gets a new recall request: the loan's due date is truncated per the loan policy's recall rules, and the "Item recalled" loan notice is sent to the current borrower — but the "Recall request" *request*-notice is deliberately **not** sent (it's a creation receipt, already implied covered). When a recall request **already exists** in the queue and another recall request is moved onto that same item (e.g. via reordering), the due date must **not** be truncated again and neither notice fires a second time. A recall-notice case must distinguish "first recall on this loan" from "additional recall on an already-recalled loan" — they have different expected outcomes, not the same one twice. (cases — C2368)
29. **Any field with no data renders as a literal `-`, never blank**, across Select-item modals, Reorder-queue, and the main results table — see the dedicated note above. (cases — C680499)
30. **Batch/multi-item requesting is an edge-patron API feature** with a per-Member-tenant configurable item cap (`multiItemBatchRequestItemsValidation` / `maxAllowedItemsCount`) and Secure-tenant mediation (`mediatedRequestStatus: Pending`); write these as Postman-style API cases, not UI flows. (cases — C1292036, C1301731; refs MODPATRON-253, MODREQMED-161)
31. Request type availability is also governed by the request policy: a type the policy disallows is not created; via API the error is a 422 with error body, never a 500. (Karate `requests.feature` policy scenarios; CIRC-2104; CIRC-1944, CIRC-1964)
32. `POST /circulation/requests` requires `requesterId`, `requestType`, `requestDate`, `fulfillmentPreference`; `requestLevel`/`instanceId`/`holdingsRecordId` stay optional for backward compatibility. (mod-circulation `ramls/request.json`; Karate `requests.feature` VuFind scenario)
33. A Hold Shelf request without a pickup service point is rejected with `Hold shelf fulfillment requests require a Pickup service point`. (en.json `errors.holdShelfRequestsRequirePickupServicePoint`)
34. The pickup SP must be allowed by the request policy for the request type; a disallowed SP is rejected for item- and title-level requests, and `GET /circulation/requests/allowed-service-points` returns only valid SPs. (Karate `requests.feature` + `tlr-requests.feature`; en.json `errors.requestPickupServicePointIsNotAllowed`)
35. A Delivery request must save even when the request policy restricts pickup SPs for that request type. (CIRC-2122)
36. A service point that is deleted or stops being a pickup location is removed from request policies' allowed SPs. (Karate `requests-extended.feature`)
37. Patron group limits (max items charged out, max overdue recalls, max outstanding fee/fine balance, recall overdue by max days, max lost items, max overdue items) block request creation. (Karate `requests.feature`)
38. Title-level Hold/Recall is rejected when a pageable available item exists on the instance; a title-level Page needs an item. (en.json `errors.holdAndRecallTlrNotAllowedPageableAvailableItemFound`, `errors.cannotCreatePageTlrWithoutItemId`; docs "How FOLIO decides which item will fill a title-level request")
39. TLR item choice: available item at a location whose primary SP is the pickup SP first, then the nearest location by Institution > Campus > Library > Location; with no available item the TLR becomes a Hold or Recall. (docs)
40. A title-level Recall applies to the loan with the earliest due date; the returned item goes to the first open request in the title queue, not necessarily the recall. (docs)
41. A TLR can be placed when one item of the instance is `Restricted` and other items are available. (CIRC-2439)
42. TLR Recall is allowed for items `In process`, `In transit`, `On order` and not allowed for `Aged to lost`, `Declared lost`, `Claimed returned`. (Cypress C375939, C375941, C375942, C375946, C375947, C375949)
43. A requester cannot place a TLR when they already have an item-level request on the same instance, and vice versa. (Cypress C350685, C350686; en.json `errors.itemOfThisInstanceAlreadyRequested`, `errors.instanceHasAlreadyBeenRequested`)
44. A title-level Page cannot be moved to the same item it is already on. (en.json `errors.movingRequestToTheSameItem`)
45. A moved request goes to the end of the target item's queue. (Cypress C6652)
46. "Item recalled" notice is sent for every recalled item of a title, not only for the item first in the title queue. (CIRC-2168)
47. An item without a barcode can be selected when creating a request (incl. TLR item selection). (UIREQ-1275, UIREQ-1298; Cypress C745/C10956)
48. When a proxy creates a request, the sponsor's patron group and delivery address are shown, not the proxy's. (UIREQ-1135, UIREQ-1132, UIREQ-1133)
49. For use at location needs mod-circulation `ENABLE_FOR_USE_AT_LOCATION=true` (default false). (CIRC-2457; ModuleDescriptor launchDescriptor)
50. New request shows `Loan type` in Item information and `For use at location` in Request information when the loan policy for this requester + item is UAL — it depends on both, and works for Available items without a loan. (UIREQ-1282, UIREQ-1326, UIREQ-1327; en.json `loanType`, `forUseAtLocation`)
51. Request details show `Loan type` for Page and Hold requests. (UIREQ-1328; CIRC-2441)
52. A UAL request never offers Delivery, even if the requester's preference is Delivery. (UIREQ-1329; Confluence 1779171329)
53. Anonymized request: requester `Anonymized`, no link, barcode `-`, proxy box hidden (details); list `Anonymized` / `-` / `-`; CSV fields empty. (UIREQ-1313, UIREQ-1314)
54. Deleted requester (not anonymized): `Unknown user`, no link, barcode `-`, in any request status. (UIREQ-1312; en.json `errors.user.unknown`)
55. Only Closed-* requests can be anonymized; anonymization removes requester id/name/barcode, proxy data, and delivery address (Delivery requests), sets `userBarcode` `-` on related circulation-log entries and adds a log entry with action `anonymizeRequest`; open requests are left unchanged. (CIRC-2292; API `POST /request-anonymization/{requestId}` CIRC-2364)
56. CSV export still works when the user who last printed a pick slip was deleted; `Printed` then holds only the date. (UIREQ-1305; Karate `print-events-extended.feature`)
57. `# Copies`, `Printed`, and the `Print status` filter exist only when "Enable view print details (Pick slips)" is on; `# Copies` = number of pick-slip prints. (UIREQ-1119, UIREQ-1121; docs)
58. Print details are logged (request, user, date/time) on `Print pick slips {sp}`, `Print selected pick slips {sp}`, and `Single print`, and `# Copies` increments for every print action. (UIREQ-1122; UIREQ-1261/CIRC-2265)
59. The `Print status` filter (`Printed` / `Not printed`) persists across apps, and CSV export works with it applied. (UIREQ-1119)
60. Pick slips print grouped/sorted by effective location (ascending), then shelving order, with title as fallback when there is no shelving order. (CIRC-1889)
61. Search slips cover only Hold requests in `Open - Not yet filled` for items whose effective location belongs to the signed-in SP; the option is greyed out when nothing matches. (docs)
62. Hold shelf clearance report rows: item `Awaiting pickup` AND request `Closed - Cancelled` or `Closed - Pickup expired` AND item queue empty or top request not `Open - Awaiting pickup`; report is per signed-in SP. (docs)
63. Search box matches request id, requester id, requester barcode (starts-with), title, instance id, item barcode (starts-with), item id, normalized ISBN (with or without hyphens), call number. (`src/utils.js`; UIREQ-1355 / C343043; UIREQ-1232)
64. Retrieval service point: optional column, sortable with pagination, filter (ECS and non-ECS), and included in CSV. (UIREQ-1188–1191, UIREQ-1294; Cypress C627236, C627454)
65. CSV export includes the Request UUID. (UIREQ-1264)
66. ECS with mod-tlr: Duplicate, Move, and Proxy are hidden; Actions is hidden when empty; edits only from the originating tenant. (UIREQ-1127, UIREQ-1128, UIREQ-1185, UIREQ-1088)
67. Intermediate requests in Central offer only `Reorder queue`. (UIREQ-1254)
68. `Requests: View` alone can print pick/search slips; `Requests: View, create` alone can use title look-up. (UIREQ-1217; UIREQ-1296; package.json permissionSets)
69. A request whose instance/item data was erased opens without crashing and shows `-` (not a link) for the title. (UIREQ-1365)
70. `Year` shows all years when the instance has several. (UIREQ-1363)

---

## Authoring style (measured 2026-07-23)

Requests cases split ~53% `Functional` / 45% `Other` (pick per scenario; API/edge-patron flows tend `Functional`), median ~5 steps, `User Journey` flag ~1% (leave `No`). Core UI request flows are compact atomic cases; batch/edge-patron and ECS/mediated flows are API cases with method/URL/header/body blocks. Preconditions group related facts per numbered item (TLR settings in one item, shared-instance+holding in another).

---

## Known Gaps and Verification Notes

- Focus-release slice is present, but Umbrellaleaf-tagged cases were not found in this subtree during this run; recent signal comes primarily from Trillium, Sunflower, and Ramsons.
- TestRail toast extraction for Requests is noisy because several cases assert translation tokens or backend field names instead of final rendered text. Verify toast wording in tenant before writing strict expected results.
- (2026-06-17 build) GitHub and Jira enrichment was limited then; the 2026-11-24 refresh added Jira UIREQ/CIRC, ui-requests/mod-circulation source, Karate, Cypress and docs — rules without a Jira/source key (1–14, 16–18) still rest on docs + cases.
- Circulation-settings capability names related to request-printing settings should be re-validated in Eureka before using them in new cases.
- **TestRail not queried in the 2026-11-24 refresh (bridge disconnected).** Case IDs added in this refresh come from stripes-testing spec titles; Umbrellaleaf-tagged cases and cases created after 2026-11-07 are not reflected.
- **Label conflicts:** docs.folio.org says "Export search results to CSV" and "Export hold shelf clearance report for [your service point]"; en.json has `Export search results (CSV)` and `Hold shelf clearance report {currentServicePoint} (CSV)`. Confluence 1779171329 says "Use at location"; en.json `forUseAtLocation` = `For use at location`. Use the translation and confirm in the tenant.
- **Inactive-patron modal body** `Inactive users cannot make requests.` is not in ui-requests en.json (backend/other module) — keep the case-sourced text (C1385311) but verify.
- **Open bugs/stories — do not assert as fixed:** UIREQ-1369 (duplicating a request copies printed details — they should be cleared; Trillium ECS BF); UIREQ-1370 / CIRCSTORE-693 ("View requests in queue" groups requests of different titles/items — proposal to show `-` position for virtual records); UIREQ-1307 (restore barcode link in Central); UIREQ-1319 (suppress search-slips call when disabled); UIREQ-1228 (print details for search slips — only pick slips are logged today); UIREQ-1359 (UI capability for manual request anonymization); CIRC-2365 (anonymize a list of requests API).
- **TLR on instances with > 20 items:** CIRC-2389 spike (not all items evaluated for fulfilment, CIRC-2354) is closed as Done but no fix is linked in this refresh — verify before writing a "large instance" TLR case.
- **Request anonymization automation:** settings storage (CIRCSTORE-629) and timer exist; the Settings UI form (UICIRC-1202) and the exact setting labels were not verified here — see circulation-settings.md.
- **Won't Do — never write cases for:** "Open - Not on shelf" status and its filter/action (UIREQ-956/957/958); "Reset to Open - Not yet filled" (UIREQ-584); Move for `Open - In transit` (UIREQ-577); move above a page (UIREQ-301); due date in request list (UIREQ-428); custom item statuses in requests (UIREQ-475/480); required-additional-info flag per cancellation reason (UIREQ-777); staff note on cancel (UIREQ-470); pick-slip warning pop-up (UIREQ-1120).
- **Not verified in source:** "Title-level requesting cannot be disabled while open title-level requests exist" (rule 11) and the multi-volume TLR limit (rule 12) come from docs only.

> Random spot-check (2026-07-22): picked one fresh uncited case at random (C680499) — no prior rule existed for empty-field display, so this was a genuine blind spot rather than a re-confirmation. Added the dedicated note and Key Business Rule 29.


---

## Change Log (delta of 2026-11-07 folded in on 2026-11-24)

> The 2026-11-07 delta section was merged into the sections above; nothing below overrides them.

- Requests-relevant Jira from that delta now live in the main sections: UIREQ-1365 (erased data, dash link), CIRC-2697 / CIRC-2700 (batch queue positions), CIRC-2538 (ECS pick-slip SP), CIRC-2664 (duplicate hold-shelf-expiration notice), CIRC-2639 (item/title missing in email notices for ECS and mediated requests — verify in patron-notices.md).
- Non-Requests items from that delta (loan-policy automatic renewals, notice-policy reordering, in-house use at check-in, Vancouver time zone, circulation log, staff-slip names) belong to circulation-settings.md, patron-notices.md, check-in.md, loans.md, circulation-log.md and were dropped here.
- TestRail cases created 2026-06-17..2026-11-07 (from the delta): C1574456 Patron account endpoint returns information about failed batches (MODPATRON-284, MODPATRON-291, EDGPATRON-211); C1543956 Create batch ECS and Mediated requests for more than 10 items and same patron (MODPATRON-283, MCBFF-216, MCBFF-224, MODPATRON-289); C1474739 Create Mediated batch request for Secure patron who has another request in "Open - Awaiting pickup" status (MCBFF-206); C1395025 Hyperlink is removed for cancelled Inn-Reach requests (UIREQ-1365).
- 2026-11-24: full source refresh — API surface, settings, capability mapping, UAL, anonymization, ECS notes, automated coverage, rules 31–70; corrected `# Copies` meaning, Actions labels, rule 15.
