# FOLIO Mediated Requests — Business Logic Context

> Built: 2026-07-21 · Delta-refreshed: 2026-11-07 · Refreshed: 2026-11-24 (remediated 2026-11-25) · Release: Umbrellaleaf R2 2026 (in development — master `ui-requests-mediated` 4.1.0 IN_PROGRESS, `mod-requests-mediated` 1.4.0-SNAPSHOT; last released: Trillium R1 2026 = ui 4.0.1 / mod 1.3.3) · Builder: build-app-context (refresh mode, full merge)
> Sources (2026-11-24 refresh): Jira UIREQMED (≈100 issues) + MODREQMED (≈100 issues), 54 read in detail; GitHub `ui-requests-mediated` (translations/ui-requests-mediated/en.json — identical to en_US.json at HEAD, package.json, CHANGELOG.md), `mod-requests-mediated` (descriptors/ModuleDescriptor-template.json, swagger.api schemas, ErrorCode.java, RequestEventHandler.java, MediatedRequestsRepository.java, README.md, NEWS.md); `folio-integration-tests` folio-ecs-circulation (vega/mediated-requests); `stripes-testing`; TestRail suite 21 sections 32570 (74 cases), 57628 (10), 61629 (5), 74667 (2); Confluence (Secure notices design page 815431766, "Workflows (Both non-Congressional and Congressional)" 803962948). docs.folio.org has no Mediated requests page.

> Built 2026-07-21 (browser-based); **corrected 2026-07-21 same day** after re-verification found real 2025 TestRail coverage that was missed on first pass. **Exact UI Texts confirmed against `folio-org/ui-requests-mediated/translations/ui-requests-mediated/en_US.json`** (re-checked 2026-11-24 against `en.json` HEAD: 211 keys, no differences between en.json and en_US.json). TestRail: suite 21 section 32570 "Mediated requests" (subsections 57628 "oLOAN", 61629 "Mediated request notices", 74667 "Mediated request patron blocks") — contains 15+ cases created in 2025 alone (releases Sunflower=19/Trillium=20); this is NOT a thin-coverage area, it was simply not pulled when this file was first built from GitHub translations only. Business rules below now include both translation-derived and live-case-confirmed content (marked accordingly).

---

## What is Mediated Requests

Mediated Requests lets library staff place and manage requests **on behalf of patrons** (a mediated / "secure" workflow, typically used with a **secure tenant** holding sensitive patrons). A staff-created request starts as **New - Awaiting confirmation** and must be reviewed and **confirmed** (which creates the real request in the Requests app) or **declined**. The app also drives the physical hand-off: **Confirm item arrival** and **Send item in transit** activities move the item toward the requester's pickup service point.

Jira/repo: `UIREQMED` / `MODREQMED`, `folio-org/ui-requests-mediated`, `folio-org/mod-requests-mediated` (Vega team). Related BE flows live in `mod-circulation-bff` (MCBFF — confirm, check-in/out, declare lost), `mod-tlr` (MODTLR — ECS TLR / fake patrons) and `mod-dcb` (MODDCB — circulation items).

**Where the app runs:** the Mediated requests app is displayed and usable only in the **Secure** tenant; it is not shown in the Central tenant or any non-Secure member tenant (C1434658, BF-1522; UIREQMED-75 comment: "Mediated app is not working" in Central). The BE identifies the Secure tenant from env var `SECURE_TENANT_ID` (mod-requests-mediated README.md) — Kafka-driven status changes are applied in that tenant only (folio-integration-tests `vega/mediated-requests/mediated-requests.feature` comment).

---

## Key Terms

| Term | Definition |
|---|---|
| Mediated request | A request staff create for a patron that requires confirmation before becoming a real request. |
| Secure tenant | Tenant holding the mediated patrons/records the workflow protects. |
| Confirm | Approve a `New - Awaiting confirmation` mediated request → creates the actual request. |
| Decline | Reject a mediated request → `Closed - Declined`. |
| Confirm item arrival | Activity confirming the paged item has arrived for approval. |
| Send item in transit | Activity routing the item onward (e.g. to the pickup service point / to be checked out). |
| Interim service point | Service point used in the mediation/confirmation hand-off. |

---

## App structure — three activities

Filter pane `Select activity` switches between:
1. **Mediated requests** (`Mediated requests activities`) — the request list + New/Edit/Decline.
2. **Confirm item arrival** — `Scan items` pane to confirm arrivals.
3. **Send item in transit** — `Scan items` pane to route items onward.

---

## Status lifecycle [confirmed]

```
New - Awaiting confirmation
   ├─ Edit & confirm → (real request created) → Open - Not yet filled
   └─ Decline → Closed - Declined

Open - Not yet filled
   → Open - In transit for approval → Open - Item arrived
   → Open - In transit to be checked out
   → Open - Awaiting pickup  /  Open - Awaiting delivery
   → Closed - Filled   (or Closed - Cancelled)
```

Confirmed status values: `New - Awaiting confirmation`, `Open - Not yet filled`, `Open - In transit for approval`, `Open - Item arrived`, `Open - In transit to be checked out`, `Open - Awaiting pickup`, `Open - Awaiting delivery`, `Closed - Filled`, `Closed - Cancelled`, `Closed - Declined`. Same 10 values in the BE enum `status` (mod-requests-mediated `swagger.api/schemas/mediatedRequest.yaml`), in en.json keys `mediatedRequestStatus.*`, and in the filter list of C494032 / UIREQMED-37. In the API the status is also split into `mediatedRequestStatus` (`New` / `Open` / `Closed`) + `mediatedRequestStep`. Obsolete filter values `Open - In progress` / `Open - In transit` in UIREQMED-25 were replaced by UIREQMED-37 — do not use them.

### Transitions and what triggers them [source-confirmed]

| From → To | Trigger | Source |
|---|---|---|
| (none) → `New - Awaiting confirmation` | `Save & close` on New mediated request form (POST `/requests-mediated/mediated-requests`) | UIREQMED-7/22, C584493, C584498 |
| `New - Awaiting confirmation` / form → `Open - Not yet filled` | `Confirm` (POST `/requests-mediated/mediated-requests/{id}/confirm`, called via `circulation-bff.confirm-mediated-request.execute`) — creates the circulation request(s) and fills `Confirmed request` | UIREQMED-48/49, C584500, Karate `mediated-requests.feature` |
| `New - Awaiting confirmation` → `Closed - Declined` | `Decline` — BE rejects decline for any other status with validation error `Mediated request status should be 'New - Awaiting conformation'` (typo is in the code) | UIREQMED-42, `MediatedRequestActionsServiceImpl.declineRequest` |
| `Open - Not yet filled` → `Open - In transit for approval` | Check in of the item in the lending tenant: primary circulation request goes `Open - Not yet filled` → `Open - In transit` (routed to the Interim service point) | `RequestEventHandler.java`, C1061228, C957383 |
| `Open - In transit for approval` → `Open - Item arrived` | Mediated requests > `Confirm item arrival` — BE only finds requests in step `In transit for approval` | `MediatedRequestsRepository.java`, Karate comment |
| `Open - Item arrived` → `Open - In transit to be checked out` | Mediated requests > `Send item in transit` — BE only finds requests in step `Item arrived`; prints the `Transit (Mediated requests)` staff slip | `MediatedRequestsRepository.java`, UIREQMED-40 |
| `Open - In transit to be checked out` → `Open - Awaiting pickup` / `Open - Awaiting delivery` | Check in at the pickup SP (Secure tenant): circulation request `Open - In transit` → `Open - Awaiting pickup` / `Open - Awaiting delivery` | `RequestEventHandler.java`, MODREQMED-127 |
| `Open - Awaiting pickup` / `Open - Awaiting delivery` → `Closed - Filled` | Check out in the Secure tenant (circulation request → `Closed - Filled`) | `RequestEventHandler.java`, C1061228 |
| any status → `Closed - Cancelled` | The confirmed circulation request is cancelled in the Requests app; Open mediated requests cannot be cancelled in the Mediated requests UI | `RequestEventHandler.java`, UIREQMED-37 note, C630441/C630442 |

Only events for the **primary** ECS request (or a non-ECS request) are processed; secondary/intermediate request events are ignored (`RequestEventHandler.java`). Each transition fires only when both the old circulation status and the current mediated step match — e.g. checking in at the pickup SP before `Send item in transit` does not move the mediated request to `Open - Awaiting pickup`.

---

## Exact UI Texts [all confirmed — ui-requests-mediated en_US.json]

### New / Edit mediated request form
- Pane titles: `New mediated request` / `Edit mediated request`.
- `Create title level request` checkbox.
- Accordions: `Title information`, `Item information`, `Requester information`, `Request information`.
- Inputs: `Instance HRID or UUID` (placeholder `Enter HRID or UUID`, lookup `Title look-up`); `Item barcode` (placeholder `Scan or enter item barcode`); `Requester barcode` (placeholder `Scan or enter requester barcode`, lookup `Requester look-up`).
- Request fields: `Request type` (`Select request type`; options `Page` / `Hold` / `Recall`), `Request status`, `Patron comments`, `Fulfillment preference` (`Hold Shelf` / `Delivery`), `Pickup service point` (`Select pickup service point`), `Delivery address`.
- Buttons: `Enter`, `Cancel`, `Confirm`.

### Form errors [confirmed]
| Condition | Message |
|---|---|
| No instance selected | `Please select an instance and hit enter` |
| No item selected | `Please select an item and hit enter` |
| Item barcode invalid | `Item with this barcode does not exist` |
| No requester | `Please fill out requester information to continue` |
| Requester barcode invalid | `User with this barcode does not exist` |
| Instance HRID/UUID invalid | `Title with this HRID or UUID does not exist` |

### Toasts [confirmed]
| Event | Text |
|---|---|
| Mediated request saved | `Mediated request has been successfully saved` |
| Request created (on confirm) | `Request has been successfully created for {requester}` |
| Save failed | `Mediated request cannot be successfully saved` |
| Create failed | `Request was not successfully created` |
| CSV report generating | `A CSV results report is being generated. This may take some time for larger result sets.` |

### List, details & actions [confirmed]
- List columns: `Mediated request date`, `Title`, `Item barcode`, `Effective call number string`, `Mediated request status`, `Requester`, `Requester barcode`. Action menu: `New mediated request`, `Export results (CSV)`.
- Details pane: `Mediated request details`; action menu `Edit & confirm`, `Decline`. Accordions: `Mediated request information`, `Requester information`, `Title information`, `Item information`. Fields: `Mediated request type`, `Mediated request status`, `Mediated request level` (`Item`/`Title`), `Confirmed request`, `Patron comments`; link `View details in Requests`.
- Decline modal: title `Confirm Mediated request decline`; message `<b>{title}</b> will be <b>declined</b>`; `Confirm` / `Back`.

### Search by UUID + CSV export (confirmed live, C844200, 2025)
The `Select activity` pane's search box accepts the **mediated request UUID** directly (not just barcode/name text). **Correction 2026-11-24:** the UUID search is an **exact** match only (UIREQMED-112 note: "The Mediated requests UUID search is an Exact search"); a partial UUID or a UUID with extra characters returns the no-results message, not a 400 error (MODREQMED-167). The text fields are "starts with" searches: Title, Item barcode, Requester barcode, Effective call number string (prefix + call number + suffix) (UIREQMED-26, C503170–C503173); search is case-insensitive (C648521, MODREQMED-17) and accent-insensitive (C663264). `Reset all` clears the result table entirely (not just the filters). No-match message (exact team wording): `No results found for "{search value}". Please check your spelling and filters.` — this fires both for a *partial* UUID and for a value that matches nothing. `Actions > Export results (CSV)` downloads a CSV whose row count must equal the on-screen result count — a case should verify the file's row-for-row content, not just that a download happened.

### Three-tenant structure is the norm, not the exception (confirmed live, C692196, 2025)
A realistic mediated-request scenario typically spans **three** tenants at once, not just "a secure tenant": a **Central** tenant where `Settings > Circulation > General > Consortium title level requests (TLR) > Enable consortium title level requests (TLR)` must be turned on first; the **Secure** tenant (e.g. "University") where the mediated patron/requester lives and where staff work in the Mediated Requests app; and a **member** tenant (e.g. "College") holding the shared Instance/item actually being requested. Preconditions for a new case should name all three roles explicitly, not just "secure tenant" — and `ECS Enabled = true` is the norm for this app's cases, not an exception.

### Creating a request can go straight to Confirm (confirmed live, C692196, 2025)
The `New mediated request` form itself exposes a `Confirm` button (enabled once item + requester + request type + pickup service point are filled) — a staff user with confirm capability can create and confirm in one flow, landing directly on `Open - Not yet filled` without stopping at `New - Awaiting confirmation`. The two-step "create then separately Edit & confirm later" flow documented below is the path for a request created by staff **without** confirm capability (or deliberately saved for a second reviewer) — both paths are real and should be covered as distinct scenarios, not assumed to be the same case shape.

### Confirm item arrival / Send item in transit [confirmed]
- Panes `Confirm item arrival` / `Send item in transit`; `Scan items` main section; input placeholder `Scan or enter item barcode`; `Enter`; empty `No items have been entered yet`.
- Error modals: `Item not confirmed as arrived` / `No "Confirm item arrival" transaction could be found for this item`; `Item not sent in transit` / `No "Send item in transit" transaction could be found for this item`; `Close`.
- Confirm-item list columns: `Arrival date`, `In transit date`, `Title`, `Item barcode`, `Effective call number string`, `Mediated request status`, `Requester`, `Requester barcode`.

### Patron Notices for Mediated Requests [confirmed live — C692073, C692119; resolves prior Known Gap]

- Non-TLR mediated requests use the SAME notice mechanism as ordinary Requests: templates + a policy configured in the **Secure** tenant's own Settings > Circulation > Patron notice templates/policies (Category = Request), keyed to the identical triggering events (`Page request`, `Awaiting pickup`, `Cancel request`). Confirmed firing correctly across the full mediation lifecycle (create → Confirm item arrival → Send item in transit → Awaiting pickup → Cancel), even though the request physically spans Central/Member/Secure tenants.
- TLR mediated requests instead use the dedicated **"TLR Patron notice templates"** settings page (Secure tenant) with exactly two slots: **Confirmation notice** and **Cancellation notice** — this mirrors circulation-settings.md's TLR notice concept, now confirmed specifically for the Mediated-requests/Secure-tenant context.
- Cancelling the REAL circulation Request from ANY tenant's Requests app (not just Secure) still correctly fires the Secure tenant's configured Cancel/Cancellation-notice template — notice wiring stays anchored to the Secure tenant's policy regardless of where the cancel action itself was performed.

### Patron Blocks in Mediated Requests [confirmed live — C831971, C831973; resolves prior Known Gap]

- **"Requests" patron block** fires the standard `Patron blocked from requesting` popup the moment a blocked patron's barcode is entered (Enter) into "Requester barcode" on the New-mediated-request form — Override opens the same "Override patron block" modal (required Comment) used elsewhere; after Save & close, the New-request form remains open with all previously entered values retained, letting staff finish and save/confirm normally. Confirmed to work identically whether the requested instance is Shared (lives in a Member tenant) or Local (lives in the Secure tenant itself).
- **"Borrowing" patron block** fires when checking out the item once it arrives via the mediation workflow (Check out app) — same `Patron blocked from borrowing` popup → Override → Comment → checkout completes as `Checked out through override`.
- These are two INDEPENDENT gates: overriding the Requests block at creation time does NOT carry over to checkout — a Borrowing-blocked patron must be separately overridden again at the Check-out step even for the very same mediated-request journey.

---

## Secure Patron Data Masking [confirmed live — C627409, C856508; previously undocumented]

- Once a Mediated Request is confirmed, the resulting REAL circulation Requests visible in the **Central** and **Member** tenants display the requester under a fixed fake identity: name `Patron, Secure`, barcode of the exact form `securepatron_{uuid}` — the real patron's identity is never exposed outside the Secure tenant.
- On these masked Request-details pages (viewed from Central/Member), the entire `Actions` menu is suppressed — staff in those tenants cannot Cancel, Edit, or otherwise act on a secure-patron request at all; only Secure-tenant staff (or the mediated-request-origin flow) can act on it.
- The same masking applies inside the **Reorder queue** view: Requester/Requester-barcode columns show the masked values when viewed from Central/Member, but show the REAL name/barcode when viewed from inside the Secure tenant itself.
- **Known cross-tenant staleness (accepted, MODTLR-150)**: reordering a queue from Central correctly updates "Position in queue" consistently in the Member/Data tenant's own view, but the SAME queue viewed from the Secure tenant does NOT reflect the updated order — a reorder-queue case covering all three tenants should expect this specific lag, not treat it as a failure.

## "Congressional Loan" — Cross-Tenant Loan Lifecycle After Checkout [confirmed live — C812848, C825272; new terminology]

- Once a mediated request completes checkout, the resulting loan is referred to in real cases as a **"congressional loan"** — a real loan whose patron/loan record lives in the Secure tenant but whose item physically resides (and is checked in/out) across Member/Central tenants too. Recognize this term when it appears in stories/cases for this area.
- **Declare lost** must be performed from the SECURE tenant's Loan details (where the real patron/loan lives); the SAME loan is then visible read-only from the Member ("data") and Central tenants' own Loan-details views, each showing `Declared lost` with its OWN tenant-specific Lost item fee policy NAME — the same cross-tenant loan can have differently-named lost-item-fee policies applied per tenant — but the `Declare lost` action itself is disabled everywhere except the Secure tenant.
- Checking in a declared-lost congressional loan typically requires **multiple check-in passes across different tenants** (e.g. Member/Data tenant → Secure tenant → Data tenant again) before the item finally reaches `Available`, with intermediate `In transit` states and confirmation popups at each hop — reflecting genuine multi-tenant physical routing, not a bug.
- **Claim returned** must likewise be actioned from the Secure tenant (where "Resolve claim" appears), but the "Found by library" CHECK-IN resolution can be performed from a different tenant's Check-in app. A known timing quirk (tracked as MCBFF-158, cited directly in the case) means the Secure tenant's own view of the loan may lag in showing it as closed immediately after a different tenant's check-in resolves it — expect (don't fail on) this specific lag when writing/running such a case.
- **Congressional-loan mechanics generalize beyond the first two sampled cases** (confirmed live, C895640, blind-generate/diff validation round, 2026-07-22): declaring a congressional loan lost with a **$0 Set-cost Lost item fee policy** auto-closes the loan immediately at the Declare-lost confirm step itself (`Closed loan` action + `Lost and paid` item status appear together in the SAME action, `Declare lost` button becomes disabled, `Fees/fines incurred` stays blank) — the exact same $0-auto-close pattern already documented in `loans.md` for ordinary loans is confirmed to generalize unchanged to the Secure-tenant congressional-loan case. **Central tenant has no record of oLOAN materials at all** — searching Central's own Circulation log for the item's barcode returns zero results, because the request/loan for oLOAN (congressional-loan) materials literally does not exist as a Central-tenant record, not merely a masked/hidden one. **Check-in of a declared-lost congressional item from a non-Secure tenant suppresses `Loan details` and `Patron details`** from that check-in result row's ellipsis actions menu — a third confirmed masking surface alongside the Requests-details Actions-menu suppression and the Reorder-queue masking already documented above.

- **Claim returned then check-in (confirmed live, C895642)**: after `Claim returned` in the Secure tenant (`Confirm item status: Claimed returned` modal, then `Resolve claim` replaces the button), a check-in at a non-DCB service point shows the `Check in claimed returned item?` popup; choosing `Returned by patron` sets the item `In transit` and the loan appears under `Closed loans` in the Secure tenant; a second check-in at the item's effective-location SP makes it `Available`. Central Circulation log shows no actions for the oLOAN loan. Cited refs include MCBFF-158, MODTLR-296, MODDCB-292/293.
- **Declare lost after Claim returned (confirmed live, C895657)**: `Resolve claim` > `Declare lost` with a $0 Set-cost Lost item fee policy behaves like C895640 (`Declared lost` + `Closed loan` actions, `Lost and paid`, `Fees/fines incurred` blank, `Lost item fee policy` = the Secure tenant's policy name). Preconditions require uniquely named Lost item fee policies, circulation rule and fee/fine owners with service points in each tenant (Central and Secure). Check-in of the lost item at a non-DCB SP now shows a confirmation popup (after MCBFF-136, UICHKIN-488) before `In transit`.
- **oLOAN Requests-app CSV (confirmed live, C1259769, UIREQ-1352)**: in the Requests app of the Secure tenant, a title search returns the mediated-origin requests and `Export search results (CSV)` includes all of them; the same search from Central shows none. The toast in that case ends with `Please be patient.` (Requests app wording; the Mediated requests app toast in en.json has no such suffix).

## Additional Confirmed Edge Cases

- **Inactive-user confirm failure** [C825435]: attempting to `Confirm` a new mediated request for a patron who has since become Inactive fails with toast `Request was not successfully created`; the form stays open; after Cancel, searches in ALL THREE tenants (Secure/Member/Central) consistently show zero open requests for that item — no orphaned partial request is left anywhere in the consortium.
- **Proxy/Sponsor UI is deliberately hidden** [C624245]: scanning a user with an active proxy/sponsor relationship into "Requester barcode" during mediated-request CREATION does NOT trigger the normal "Who are you acting as?" modal seen in Requests/Check-out — the mediated request is always created directly for the scanned patron, with no proxy-substitution path offered in this app.
- **Secure loan renewal re-applies circulation rules** [C1307996, UIU-3512]: when the patron group, material type, loan type or location of a Secure loan from a mediated request changes, the next renewal applies the new loan policy, and Loan details show it at once without a page refresh. API-only renewal and renewal-override error handling for Secure loans is covered by C1525841 and C1528161 (UIU-3537, CIRC-2632; C1525841 is also automated in Karate `mediated-requests.feature`).
- **Mediated request full flows** end-to-end (create → confirm → check-in lending → Confirm item arrival → Send item in transit → check-in pickup → check-out → congressional loan): item level C651488, title level C651489. Due date receipt for a mediated loan: C736679. Returns routing for non-oLOAN Delivery: C729206.
- **Deleted-Title edit error retains the stale value** [C594453]: if the Instance behind an existing `New - Awaiting confirmation` request is deleted after creation, opening `Edit & confirm` still shows the original (now-stale) HRID/UUID in the search box alongside the standard `Title with this HRID or UUID does not exist` error — the form surfaces the broken reference rather than silently clearing it.

## Form, list and export behaviour (from Jira ACs and bug fixes)

- **Save & close minimum:** Title/Item information + Requester information are enough to `Save & close`; removing either disables `Save & close` (UIREQMED-22 scenarios 2–3). `Confirm` needs item/title, requester, request type, fulfillment preference and pickup SP / delivery address (UIREQMED-49; C594452 `Confirm` disabled when a required field is empty).
- **Cancel with unsaved changes:** `Cancel` on the Edit form opens an `Are you sure?` modal ("There are unsaved changes", `Close without saving` / `Keep editing`) — wording from UIREQMED-22 AC, not in ui-requests-mediated en.json (stripes-components); no modal when nothing was changed (UIREQMED-64).
- **Request type disabled by circ rules:** if circulation rules allow no request type for the item/patron, `Request type` is greyed out but the mediated request can still be saved (UIREQMED-97, fixed in 2.0.3).
- **Title-level form:** `Create title level request` checkbox default follows the TLR setting "Create title level request selected by default" (C584491). Unchecking TLR after an instance is scanned opens the item list (`Select item`, `Other items in {title}` — en.json `itemsDialog.*`); items without barcode can be selected and confirmed (UIREQMED-99/100, MODREQMED-109). `Title look-up` works with only `View, create, edit` or `manage` capability (UIREQMED-116, fixed 4.0.0).
- **TLR Page before confirmation has no item:** a title-level Page mediated request in `New - Awaiting confirmation` shows no item information — expected, because no circulation request and no item match exist before confirmation (MODREQMED-208, closed "Not a bug"); en.json `mediatedRequestDetail.item.noInformation` = `There is no item information for this request.`
- **Titles longer than 255 characters** must save (verified with ≥3000 chars) (MODREQMED-136/137, C584498).
- **Missing data shows "-"** in list (Item barcode, Effective call number string), Select item modal (Barcode, Loan type), Confirm item arrival and Send item in transit lists (UIREQMED-91, C688778).
- **List:** only the `Title` cell is a link to details; clicking elsewhere on a row does nothing (UIREQMED-76). Filters persist when switching activity and back (UIREQMED-44). Clearing the search box with filters applied still shows the filtered results (UIREQMED-75). >100 results paginate (C543880).
- **Filters:** `Mediated request status` (10 values) and `Mediated request level` (`Item`/`Title`, shown only when TLR is enabled) (UIREQMED-25/37, C494032). No results from filters: `No results found. Please check your filters.`; after `Reset all`: `Choose a filter or enter a search query to show results.` (UIREQMED-25/26 AC wording — stripes SearchAndSort, not in en.json). Status/level filtering must not fail with 500 when two tenants share a location ID (MODREQMED-194/197, C958454) or for the Level filter (MODREQMED-205).
- **Export results (CSV):** inactive when there are no results; toast `A CSV results report is being generated. This may take some time for larger result sets.`; button shows `CSV report is being generated. Please wait.` while running (en.json `mediatedRequestList.actionMenu.reportPending`). CSV columns = en.json `report.*` (Mediated request type, Mediated request status, Mediated request date, Item barcode, Title, Contributor, Library, Effective call number prefix/number/suffix, Volume, Enumeration, Chronology, Item Copy, Requester name, Requester barcode, Requester patron group, Request fulfillment preference, Pickup service point, Delivery address, Requester's proxy name, Requester's proxy barcode, Patron comments) (UIREQMED-34, C553035). Export of all requests (Item + Title selected) must complete on large data (MODREQMED-176).
- **Staff notes accordion** needs Notes capabilities in addition to Mediated requests capabilities (UIREQMED-66 closed "Not a bug"; C523649).
- **Proxy hidden everywhere:** no `Who are you acting as?` modal on New or Edit, no proxy info on details, no proxy in reports (UIREQMED-59, C624245, C624246) — the CSV still has the proxy columns (en.json `report.proxy.*`), expected empty.
- **Timestamps:** Edit/Confirm update only `updatedDate` (not `createdDate`); Decline updates `updatedDate` (MODREQMED-199, C594449, C648516).
- **Send item in transit date** shows the current date/time, not `1/1/1970, 12:00 AM` (UIREQMED-104, C569599).

## Delivery requests and the Interim service point

- Confirmed requests are placed with `Hold Shelf` at the **Interim service point** (fixed ID `32c6f0c7-26e4-4350-8c29-1e11c2e3efc4`, `Constants.INTERIM_SERVICE_POINT_ID`), so the lending-tenant check-in routes the item to `Interim service point` (C957383 popup text "Route … to **Interim service point**."; C1061228). On `Confirm item arrival` the BE restores the mediated request's own fulfillment preference and delivery address onto the circulation request (`MediatedRequestActionsServiceImpl.revertConfirmedRequestDeliveryInfo`), so Delivery requests end in `Open - Awaiting delivery` (C633458, C633459, MODREQMED-127).
- The Interim SP must exist in Central and all member tenants; today it comes from mod-inventory-storage reference data, moving to automatic creation by mod-requests-mediated (MODREQMED-245, In Code Review; MODINVSTOR-1613). Missing Interim SP blocks the Secure workflow.

## Multi-item (batch) requests [new in Trillium, mod-requests-mediated 1.3.0]

- Patron (edge-patron/mod-patron) multi-item requests for secure patrons create a mediated batch with one individual request per item (MODREQMED-158/160/162/179). Batch status values: `Pending`, `In progress`, `Completed`, `Failed`; `mediatedWorkflow` = `Multi-Item request` (`schemas/batch/*.yaml`). `GET …/batch-mediated-requests/{batchId}` returns `itemRequestsStats` (total/pending/inProgress/completed/failed) (MODREQMED-164).
- Patron comments are copied to each individual request with the suffix `Batch request ID: {uuid}` (MODREQMED-158 AC).
- Max items per batch: mod-settings scope `mod-requests-mediated.manage`, key `multiItemBatchRequestItemsValidation`, value `maxAllowedItemsCount`; fallback env `BATCH_REQUEST_MAX_ITEMS` (default 50) (README.md, MODREQMED-161). Exceeding it → error 107 `The maximum number of items per batch request has been exceeded.` (ErrorCode.java).
- Two batches for the same items must not produce duplicate queue positions (CIRC-2697; Karate `batch-requests-queue.feature`). Batch requests for different tenants must be created reliably (MODREQMED-211). Coverage: C1250456, C1301731 (FAT reviews).

## API surface (mod-requests-mediated, ModuleDescriptor-template.json)

| Interface | Method + path | permissionsRequired |
|---|---|---|
| requests-mediated 2.0 | GET `/requests-mediated/mediated-requests` (CQL `query`, `limit`, `offset`) | `requests-mediated.mediated-request.collection.get` |
| | GET `/requests-mediated/mediated-requests/{requestId}` | `requests-mediated.mediated-request.item.get` |
| | POST `/requests-mediated/mediated-requests` | `requests-mediated.mediated-request.item.post` |
| | PUT `/requests-mediated/mediated-requests/{requestId}` | `requests-mediated.mediated-request.item.put` |
| | DELETE `/requests-mediated/mediated-requests/{requestId}` | `requests-mediated.mediated-request.item.delete` |
| requests-mediated-actions 1.2 | POST `/requests-mediated/mediated-requests/{requestId}/confirm` | `requests-mediated.mediated-request.confirm.post` |
| | POST `/requests-mediated/mediated-requests/{requestId}/decline` | `requests-mediated.mediated-requests.decline.execute` |
| | POST `/requests-mediated/confirm-item-arrival` | `requests-mediated.confirm-item-arrival.post` |
| | POST `/requests-mediated/send-item-in-transit` | `requests-mediated.send-item-in-transit.post` |
| requests-mediated-loans 1.4 | POST `/requests-mediated/loans/check-out-by-barcode` | `requests-mediated.loans.check-out-by-barcode.execute` |
| | POST `/requests-mediated/loans/check-in-by-barcode` (secure check-in: replaces real secure patron with Central fake patron in the response — MODREQMED-191) | `requests-mediated.loans.check-in-by-barcode.execute` |
| | POST `/requests-mediated/loans/{loanId}/declare-item-lost` (MODREQMED-146) | `requests-mediated.loans.declare-item-lost.execute` |
| | POST `/requests-mediated/loans/{loanId}/claim-item-returned` (MODREQMED-135) | `requests-mediated.loans.claim-item-returned.execute` |
| | POST `/requests-mediated/loans/{loanId}/declare-claimed-returned-item-as-missing` (MODREQMED-152) | `requests-mediated.loans.declare-claimed-returned-item-as-missing.execute` |
| requests-mediated-batch 1.0 | POST / GET `/requests-mediated/batch-mediated-requests`; GET `…/{batchId}`, `…/{batchId}/details`, `…/details` (CQL by `batchId`, `requesterId`, `itemId`, `confirmedRequestId`, `mediatedRequestStatus` — MODREQMED-190) | `requests-mediated.batch-mediated-requests.item.post` / `.collection.get` / `.item.get` / `.batch-details.collection.get` / `.details.collection.get` |

Loans endpoints forward to mod-circulation in the Secure tenant, then mirror the action to the Central fake-patron loan via mod-tlr (MODREQMED-135/146/152 "Logic"). UI calls confirm via `circulation-bff`, not directly (UIREQMED-39, package.json).

### Error codes (mod-requests-mediated `ErrorCode.java`)

| Code | Message | When |
|---|---|---|
| 100 | `Mediated request cannot be saved for inactive patron` | Save for an Inactive requester — HTTP 422, UI toast `Mediated request cannot be successfully saved`, nothing created in any tenant (MODREQMED-149, C825436) |
| 102 | `Mediated request cannot be confirmed for inactive patron` | Confirm for an Inactive requester — UI toast `Request was not successfully created` (MODREQMED-149, C825435) |
| 103 | `Mediated batch request with the given 'id' already exists.` | Batch POST with an existing batchId |
| 104 | `Invalid initial status for mediated batch request.` | Batch POST |
| 105 | `Not allowed to create Request for the given service point id.` | Batch item with a pickup SP not allowed |
| 106 | `Mediated Batch Request contains duplicate item IDs.` | Batch POST |
| 107 | `The maximum number of items per batch request has been exceeded.` | Batch over the max items |

Confirm item arrival / Send item in transit for an item with no matching mediated request → 404 (`Mediated request for arrival confirmation of item with barcode '…' was not found` / `Send item in transit: mediated request for item '…' was not found`), shown in the UI as the `Item not confirmed as arrived` / `Item not sent in transit` modals (MediatedRequestActionsServiceImpl.java, en.json `confirmItem.errorModal.*`).

## Existing automated coverage

- **Karate** (`folio-integration-tests/folio-ecs-circulation`, runner `src/test/java/org/folio/FolioEcsCirculationTests.java`; setup `vega/common/mediated-requests-consortium-setup.feature`):
  - `vega/mediated-requests/mediated-requests.feature` — create and decline item-level Page; create, edit and decline; create and confirm item-level Page; send item in transit + confirm arrival (statuses of secondary/intermediate/primary requests checked at every step); FAT-27420 item-level Page full workflow; FAT-27421 title-level Page full workflow; `@C1525841` renew-by-barcode validation errors for a loan created via mediated request.
  - `vega/mediated-requests/mediated-request-staff-slips.feature` (FAT-27423) — pick slip from Central for item-level Page; search slip for title-level Hold; hold/transit/request delivery slips provisioned in printing tenants.
  - `vega/mediated-requests/batch-requests-queue.feature` — two batches for the same three items keep unique queue positions before/after checkout.
  - Not covered by Karate: batch error codes 103–107, inactive-patron errors 100/102, loans endpoints (declare lost / claim returned / missing), search/filter CQL, CSV export.
- **Cypress** (`stripes-testing`): only an FSE API smoke test `cypress/e2e/fse/api/mediated-requests-api.cy.js` (TC195956, GET `requests-mediated/mediated-requests` returns 200 with `mediatedRequests`) and command `cypress/support/api/mediated-requests.js`. No UI e2e specs or fragments for the Mediated requests app.
- **TestRail**: all 74 cases in section 32570 have the same `custom_automation_type` value (2); the automation type was not pulled for subsections 57628/61629/74667.

---

## Capability Sets (Eureka) [confirmed — exact strings copied from the Capability sets list in C844200 preconditions (all 7 rows) and C692196 / C651540 (`manage` row); spelling kept verbatim: lowercase type, em dashes]

| Action | Permission (Okapi display name) | Capability/Set (Eureka) |
|---|---|---|
| View | `Mediated requests: View` | `data — UI-Requests-Mediated — view` |
| All (manage) | `Mediated requests: All permissions` | `data — UI-Requests-Mediated — manage` |
| View, create, edit | `Mediated requests: View, create, edit` | `procedural — UI-Requests-Mediated Requests-Mediated View-Create-Edit — execute` |
| View, decline | `Mediated requests: View, decline` | `procedural — UI-Requests-Mediated Requests-Mediated View-Decline — execute` |
| View, confirm | `Mediated requests: View, confirm` | `procedural — UI-Requests-Mediated Requests-Mediated View-Confirm — execute` |
| Confirm item arrival | `Mediated requests: Confirm item arrival` | `procedural — UI-Requests-Mediated Requests-Mediated Confirm-Item-Arrival — execute` |
| Send item in transit | `Mediated requests: Send item in transit` | `procedural — UI-Requests-Mediated Requests-Mediated Send-Item-In-Transit — execute` |

Cypress dictionary (`stripes-testing/cypress/support/dictionary/permissions.js`) only has `uiRequestsMediatedAll` (`ui-requests-mediated.all`, GUI name `UI-Requests-Mediated`); the capability-set strings above are NOT in `capabilitySets.js`, so their only source is the TestRail preconditions cited.

Source mapping (ui-requests-mediated `package.json` → `stripes.permissionSets`, display names = en.json `permission.*`):
- Visible sets: `ui-requests-mediated.view`, `…requests-mediated.view-create-edit.execute`, `…view-decline.execute`, `…view-confirm.execute`, `…confirm-item-arrival.execute`, `…send-item-in-transit.execute`, `ui-requests-mediated.all` (= all five procedural sets).
- Hidden building blocks (`visible: false`, not assignable alone in the UI): `ui-requests-mediated.create` (Mediated requests: Create), `ui-requests-mediated.requests-mediated.edit` (Edit), `ui-requests-mediated.requests-mediated.confirm.execute` (Confirm).
- BE permissions per set: view → `requests-mediated.mediated-request.collection.get` / `.item.get`; create/edit → `.item.post` / `.item.put` + `circulation-bff.requests.allowed-service-points.get` + `circulation-bff.requests.search-instances.get`; confirm → `circulation-bff.confirm-mediated-request.execute`; decline → `requests-mediated.mediated-requests.decline.execute`; confirm item arrival → `requests-mediated.confirm-item-arrival.post`; send item in transit → `requests-mediated.send-item-in-transit.post` + `circulation-storage.staff-slips.collection.get`.
- Actions menu per set (UIREQMED-36, UIREQMED-11, C523643–C523647): `Edit & confirm` needs `View, create, edit` or `View, confirm` (or All); `Decline` needs `View, decline` (or All); with only `View`, `Confirm item arrival` or `Send item in transit` there is no Actions menu on details. Actions are shown only for `New - Awaiting confirmation` (C523645). `New mediated request` needs `View, create, edit` (C503144). Confirm item arrival / Send item in transit activities need their own sets (C553048, C569596). No capability → app not visible (C434068).
- Other capability strings seen in cases: `data — Requests-Mediated Internal Fetch-Request-Details — manage` (C825436, BE permission `requests-mediated.internal.fetch-request-details.all` as `modulePermissions` in mod-requests-mediated `ModuleDescriptor-template.json`; the Eureka set name itself comes only from C825436 preconditions) — needed on staff users in Central/Member/Secure for cross-tenant request details.

> Real cases test each permission/capability **separately** (C844200 precondition note: "best to test each permission separately") — a capability-boundary scenario set for this app should cover each row individually, not just "manage" vs "view".

---

## Key Business Rules for Test Cases

1. A mediated request is created by staff for a patron and starts as **`New - Awaiting confirmation`** — it is NOT a real request until confirmed. (ui-requests-mediated en.json `mediatedRequestStatus.*`; UIREQMED-22; C584493)
2. **Edit & confirm** creates the actual request (`Request has been successfully created for {requester}` toast) and links it — details show `Confirmed request` and a `View details in Requests` link. (en.json `form.createRequest.success`, `mediatedRequestInformation.confirmedRequest` / `.viewDetailsLink`; UIREQMED-48/49; C584500)
3. **Decline** closes the mediated request as `Closed - Declined` after the `Confirm Mediated request decline` modal. (en.json `declineModal.*`, `mediatedRequestStatus.closedDeclined`; UIREQMED-42)
4. Request type offered (`Page`/`Hold`/`Recall`) and fulfillment (`Hold Shelf`/`Delivery`) follow the same rules as normal Requests; a title-level request uses the `Create title level request` checkbox. Option lists confirmed in en.json `form.request.type.*` / `form.request.fulfillment.*`; the claim that the rules are identical to the Requests app is only confirmed for circulation-rule-restricted request types (UIREQMED-97) and the TLR default (C584491) — everything else is (unverified), see `requests.md`.
5. The **Confirm item arrival** and **Send item in transit** activities require a matching transaction for the scanned item, else the error modal appears (`No "…" transaction could be found for this item`). (github — en.json `confirmItem.errorModal.*`; C553036, C569599)
6. Mediated request statuses extend the normal request lifecycle with mediation-specific states (`Open - In transit for approval`, `Open - Item arrived`, `Open - In transit to be checked out`). (en.json `mediatedRequestStatus.*`; mod-requests-mediated `swagger.api/schemas/mediatedRequest.yaml`; C494032)
7. Separate capability sets gate create/edit vs confirm vs decline vs the two item activities — a user may see mediated requests but lack confirm/decline. Confirmed concretely: a user with ONLY `Mediated requests: View, decline` opens a `New - Awaiting confirmation` request's details and the `Actions` menu shows exactly one option, `Decline` — no `Edit & confirm` (that requires the separate View-create-edit or View-confirm capability). Each of the 7 capability rows in the table above gates its own Actions-menu entry independently; don't assume "decline visible" implies "confirm visible" or vice versa. (package.json `stripes.permissionSets`; C523647; C844200)
8. Notes: staff notes can be attached to a mediated request (`New staff note` / `Staff notes`); the accordion needs Notes capabilities in addition to Mediated requests capabilities. (en.json `notes.staffNotes`, `notes.newStaffNote`; UIREQMED-66; C523649)
9. A realistic scenario spans **three tenants** (Central config + Secure requester tenant + member-tenant shared item) with `Enable consortium title level requests (TLR)` active in Central as a precondition — this app is ECS-by-default, not ECS-as-a-special-case. (cases — C692196, 2025)
10. The `New mediated request` form can go straight to `Confirm` in one flow (landing on `Open - Not yet filled` directly) when the creating staff user has confirm capability — the separate "create, then later Edit & confirm" path is for staff without that capability or a deliberate two-reviewer flow. Both are real, cover both. (cases — C692196, 2025)
11. The result list search accepts the mediated request **UUID** (exact match only — a partial UUID returns the no-results message) in addition to text fields, and supports CSV export whose row count must match the on-screen result count. (cases — C844200, 2025)
12. Cancelling the confirmed circulation Request cascades one-way to close the originating Mediated Request as `Closed - Cancelled` too — for both Item-level and Title-level requests. (cases — C630441)
13. Requester identity is masked outside the Secure tenant (`Patron, Secure` / `securepatron_{uuid}`), and the Actions menu is fully suppressed on masked Request-details pages viewed from Central/Member. (cases — C627409, C856508)
14. A confirmed mediated request's post-checkout loan ("congressional loan") permits Declare-lost/Claim-returned actions only from the Secure tenant, while Member/Central tenants get a read-only mirrored view with the action disabled; resolution can require multiple cross-tenant check-in passes. A $0 Set-cost Lost item fee policy auto-closes the congressional loan at Declare-lost confirm itself (same rule as ordinary loans in `loans.md`), and Central tenant has literally NO record of the request/loan at all — not just a masked view. Checking in a declared-lost congressional item from outside the Secure tenant also suppresses `Loan details`/`Patron details` on that check-in row's actions menu. (cases — C812848, C825272, C895640)
15. Requests and Borrowing patron blocks are independent gates in the mediated-request flow — overriding one does not carry over to the other, even for the same patron/item journey. (cases — C831971, C831973)
16. Proxy/Sponsor substitution is never offered during mediated-request creation, regardless of whether the scanned patron has an active proxy relationship. (cases — C624245)
17. The Mediated requests app is available only in the Secure tenant — not in Central and not in any non-Secure member tenant. (C1434658, BF-1522)
18. `Decline` is accepted only for `New - Awaiting confirmation`; the BE returns a validation error for any other status. (UIREQMED-42; mod-requests-mediated `MediatedRequestActionsServiceImpl.declineRequest`)
19. Open mediated requests cannot be cancelled in the Mediated requests app; they become `Closed - Cancelled` only when the confirmed circulation request is cancelled. (UIREQMED-37 note; `RequestEventHandler.java`; C630441, C630442)
20. `Confirm item arrival` finds only mediated requests in `Open - In transit for approval`, and `Send item in transit` only those in `Open - Item arrived`; scanning an item in any other state shows the matching "transaction could not be found" modal. (mod-requests-mediated `MediatedRequestsRepository.java`; Karate `mediated-requests.feature`; en.json `confirmItem.errorModal.*`)
21. Mediated status follows the primary circulation request: lending-tenant check-in → `Open - In transit for approval`; check-in at pickup SP after Send item in transit → `Open - Awaiting pickup` / `Open - Awaiting delivery`; Secure check-out → `Closed - Filled`. Each step fires only from the expected previous step. (`RequestEventHandler.java`; C1061228)
22. Confirmed requests are routed through the Interim service point (Hold Shelf); the original fulfillment preference and delivery address are restored on Confirm item arrival, so Delivery requests end in `Open - Awaiting delivery`. (`MediatedRequestActionsServiceImpl.java`; C957383; C633458; MODREQMED-127)
23. `Send item in transit` prints the `Transit (Mediated requests)` staff slip and sets `Open - In transit to be checked out`; the in-transit date is the current date/time. (UIREQMED-40, UIREQMED-104; C618998, C569599)
24. Save and Confirm are both rejected for an Inactive requester (codes 100 / 102), and no fake patron or circulation request is created in any tenant. (MODREQMED-149; ErrorCode.java; C825435, C825436)
25. `Save & close` needs only title/item + requester; `Confirm` also needs request type, fulfillment preference and pickup SP / delivery address. If circ rules allow no request type, the request can still be saved. (UIREQMED-22, UIREQMED-49, UIREQMED-97; C594452)
26. Editing a request whose instance, item or requester was deleted keeps the stale value in the field with the matching `… does not exist` error. (UIREQMED-51; C594453, C594454, C594455)
27. A title-level Page request in `New - Awaiting confirmation` has no item information until confirmed (`There is no item information for this request.`). Expected behaviour, not a bug. (MODREQMED-208; en.json `mediatedRequestDetail.item.noInformation`)
28. Search: Title, Item barcode, Requester barcode and Effective call number string are "starts with", case- and accent-insensitive; mediated request UUID is exact match only, and invalid or partial UUIDs return "No results found" without an error. (UIREQMED-26, UIREQMED-112, MODREQMED-167/168; C503170–C503173, C648521, C663264, C844200)
29. `Mediated request level` filter is shown only when title-level requests are enabled; status and level filters must work without 500 errors, including when two tenants share a location ID. (UIREQMED-25; MODREQMED-194/197/205; C494032, C958454)
30. `Export results (CSV)` is disabled when there are no results; the CSV has the 23 `report.*` columns and must finish for the full data set. (UIREQMED-34; MODREQMED-176; en.json `report.*`; C553035)
31. Fields with no data show "-" in the request list, Select item modal, Confirm item arrival and Send item in transit lists. (UIREQMED-91; C688778)
32. Titles longer than 255 characters can be saved and confirmed. (MODREQMED-136/137; C584498)
33. Items without a barcode can be requested via the TLR → item list path and confirmed on the first attempt. (UIREQMED-99/100, MODREQMED-109; C584500)
34. Edit and Confirm update only `updatedDate`; Decline updates `updatedDate`. (MODREQMED-199; C594449, C648516)
35. A multi-item (batch) request for a secure patron creates one mediated request per item. The max item count comes from mod-settings `multiItemBatchRequestItemsValidation.maxAllowedItemsCount`, with fallback `BATCH_REQUEST_MAX_ITEMS` = 50; exceeding it returns error 107, and duplicate item IDs return error 106. (MODREQMED-158/161/179; README.md; ErrorCode.java)
36. Secure check-in through mod-requests-mediated replaces the real secure patron with the Central fake patron in the check-in response; transit slips printed in Central contain no secure-patron data. (MODREQMED-191; C957383; C627393, C627415)
37. Secure patron data must not appear for unauthorized staff in the Circulation log or in the Claimed returned, Overdue loans, Financial transaction detail and Refunds to process manually CSV reports. (C627400, C627401, C627402, C663276, C663277)
38. Mark-missing of a claimed-returned congressional item goes through the Secure tenant, like Declare lost and Claim returned. (MODREQMED-152; C895644)

---

## Authoring style (measured 2026-07-23)

Mediated Requests: strongly **`Func` ~94%**, median ~6 steps, `User Journey` ~0%. ECS-by-default — cases assume a Secure requester tenant + Central config, and mix UI flows with edge-patron **API** blocks (batch/mediated creation). Use `Functional`. Preconditions carry the three-tenant setup (Central TLR config + Secure tenant + Member shared item) and the capability set under test; steps walk create → confirm/decline → status, or the API request/response. Capability-boundary cases test each of the 7 capability sets individually (see the capability table).

---

## Known Gaps / Items to Verify

- [x] TestRail case coverage — **corrected**: section 32570 + subsections have substantial 2025 coverage (15+ cases just in the top section); pull more as new scenarios are needed, but do not assume thin coverage going forward.
- [x] Exact Eureka capability-set strings — **confirmed live** (C844200, C692196, 2025), see Capability Sets table above.
- [x] Secure-tenant / ECS setup specifics — **confirmed live** (C692196, 2025): Central (TLR setting) + Secure (requester + app) + member tenant (shared item), see "Three-tenant structure" above.
- [x] Mediated request notices (section 61629) and Mediated request patron blocks (section 74667) — **pulled and confirmed this round** (C692073, C692119, C831971, C831973): see "Patron Notices for Mediated Requests" and "Patron Blocks in Mediated Requests" above.
- [x] The "congressional loan" terminology and cross-tenant Declare-lost mechanics — **generalization confirmed** (C895640, blind-generate/diff validation round, 2026-07-22): a 3rd oLOAN-subsection (57628) case, sampled independently, matched the predicted $0-auto-close behavior exactly and surfaced two new confirmed nuances (Central has zero oLOAN records, not just a masked view; Check-in row actions menu is also masking-suppressed). Update 2026-11-25: C895642, C895657, C1259769, C651540 now read (bodies); still unread in 57628: C651542, C651543, C1307996 (body; only its refs/summary used). Remaining unread in 32570 (74 cases), 61629 and 74667 apart from those cited above — pull before writing cases for those specific sub-scenarios.
- [ ] MODTLR-150 (Secure-tenant queue-reorder staleness) and MCBFF-158 (Secure-tenant closed-loan display lag) are both cited as accepted/known limitations in currently-live cases — re-verify they're still open before relying on them, since either could be fixed in a future release and turn the "expected" lag into an actual regression.

Added 2026-11-24:
- [ ] **Open defects to expect, not assert as fixed:** MODREQMED-216 (Open; a request two staff edit at the same time can end with the mediated request still in `New - Awaiting confirmation` and no `Confirmed request` link while the circulation request is `Open - Not yet filled`); MODREQMED-225 (Open; title-level **Hold** mediated requests: item info missing on the `Send item in transit` slip, because mod-dcb creates the circulation item with a random ID); MODREQMED-237 (Open; the details page does not load after the requester user is deleted — TLS Trillium BF; not reproducible on Sunflower); MODREQMED-221 (Open story; patron phone token for the Transit (Mediated requests) slip). Re-check their status before writing cases for these scenarios.
- [ ] **Interim service point provisioning** is moving from mod-inventory-storage reference data to mod-requests-mediated tenant init (MODREQMED-245 In Code Review, MODINVSTOR-1613). Until it is merged, environments without reference data may lack the Interim SP, and the Secure workflow will not work there.
- [ ] **Capability-string conflicts in cases:** C1434658 preconditions write `data - UI-Mediated-Requests - view` (word order differs from `data — UI-Requests-Mediated — view` in the table above) (capability set name not confirmed — verify). C825436 uses `data - UI-Requests-Mediated - view, create, manage` and `data - UI-Requests-Mediated Requests-Mediated - edit`, which match the hidden `Create`/`Edit` permission sets rather than the visible procedural sets (capability set name not confirmed — verify). Verify the exact Eureka names in an environment before reusing them.
- [ ] **UI texts not in ui-requests-mediated en.json** (from stripes-core/components or Jira ACs only): `Save & close`, `Are you sure?` / `There are unsaved changes` / `Close without saving` / `Keep editing`, `No results found. Please check your filters.`, `Choose a filter or enter a search query to show results.`, `Search could not be processed for "…". Please check your query and try again.`, `Transit (Mediated requests)` slip name. Confirm them in the environment before asserting them verbatim.
- [ ] **Staff slip tokens:** `{{requester.patronGroup}}` was reported empty on the Transit (Mediated requests) slip (UIREQMED-40 last comment, unresolved in that ticket). Verify the current behaviour.
- [ ] **Patron notices ownership:** MODREQMED-182 comment states notices are tied to the primary circulation request, not to the mediated request; the Confluence design page 815431766 describes notices sent on request update once the fake patron is resolved to the real one (CIRC-2269 / CIRC-2639). C692095 (Hold mediated TLR, expiration notice) is still marked DRAFT.
- [ ] **Search by Mediated request tags / Type facet** are not implemented: UIREQMED-111/113/114 and MODREQMED-165 are Draft or In Refinement. Do not write cases for them yet.
- [ ] **Batch requests UI:** batch (multi-item) requests are API-only (edge-patron/mod-patron, MODREQMED-158 AC). No ui-requests-mediated strings exist for them, and MODREQMED-161 QA comment reported a 500 instead of a 4xx for the limit via `/circulation-bff/batch-requests` before the fix. Verify the error status in the environment.
- [ ] **Cypress coverage** is limited to the FSE API smoke test, and no UI fragments exist, so UI cases cannot reuse Cypress fragments.
- [ ] MODTLR-150 / MCBFF-158 (listed above) were not re-checked in Jira during this refresh.
- [ ] Rules 1, 4 and 8 were previously tagged only `(github)`; now tagged with en.json keys/Jira/C-ids, but "Request type/fulfillment follow the same rules as normal Requests" is (unverified) beyond UIREQMED-97 and C584491.

> N≥10 audit round (2026-07-22): 14 cases read (C692073, C692119, C831971, C831973, C624245, C630441, C627409, C812848, C825435, C825272, C594453, C651539, C856508, C692201). Resolved the file's last open Known Gap by pulling both the Mediated request notices (61629) and Mediated request patron blocks (74667) subsections. Surfaced substantial new previously undocumented content: Secure-patron-data masking (exact placeholder format + Actions-menu suppression + two accepted cross-tenant staleness quirks), the "congressional loan" terminology and its cross-tenant Declare-lost/Claim-returned lifecycle, the one-way cancel cascade from real Request to Mediated Request, the two-independent-gates nature of Requests vs Borrowing patron blocks, hidden proxy/sponsor UI, and the inactive-user confirm-failure cleanup guarantee. Added 5 new Key Business Rules (12-16).
> Blind-generate/diff validation round (2026-07-22): predicted (before reading) that a $0 Set-cost policy would auto-close a congressional loan the same way it does an ordinary loan per `loans.md` — confirmed exactly against C895640, which also surfaced 2 new masking nuances (Central has zero oLOAN records; check-in row actions menu suppression). Closed the file's last real open Known Gap (congressional-loan generalization); the two remaining items are external-ticket-status re-checks, not undocumented behavior.


---

## Changes since 2026-07-21 (delta refresh 2026-11-07)

> Delta, not a rebuild: sections above reflect the original build. Where this section contradicts them, **this section wins**.
> 2026-11-24 refresh: the mediated-relevant items of this delta (CIRC-2639 notices, C1434658 Secure-only app, C1525841/C1528161 secure loan renewal API) are folded into the main sections above. The other bullets concern sibling circulation areas and are kept for reference only.
> Sources: GitHub folio-org/ui-requests-mediated (translations/ui-requests-mediated/en.json, HEAD v4.0.0); Jira folio-org.atlassian.net Done issues resolved since 2026-07-21; TestRail suite 21 cases created since 2026-07-21.

### What changed
- No ui-requests-mediated commits or Done issues since 2026-07-21. Notice bug for mediated requests fixed in CIRC-2639.

### Jira — Done since build (circulation: check-in / check-out / circulation-settings / patron-notices / requests / mediated / circulation-log (CIRC, UICIRC, UICHKIN, UICHKOUT, UIREQ, UIREQMED, MODREQMED, UICIRCLOG, MODAUD) since 2026-07-21)

- UICIRC-1351 / 1387 / 1390 Story — "Automatic renewals" section on Loan policy form and page (settings + validation)
- UICIRC-1355 / 1372 Story — Reordering of patron notice policies (drag & drop)
- UICIRC-1353 Story — Shortened tokens in patron notices for text messages
- UICHKIN-515 / CIRC-2685 Story — [PoC] In-house use option in Check in (inHouseUse persisted on check-in-by-barcode)
- UICHKIN-510 Story — Check in modals: default focus on the action button, not "X"
- CIRC-2130 Story — Both Set cost and Actual cost fees/fines closed when item returned
- CIRC-2705/2706/2707 Bug — America/Vancouver time zone: due date cut-off by patron expiration; releases for Sunflower/Trillium (see Vancouver incident)
- CIRC-2664 Bug — Hold shelf expiration notice sent twice; CIRC-2679 notice not sent after aged to lost; CIRC-2639 item/title missing in email notices for ECS and mediated requests
- CIRC-2697 Bug — [LoC] batch requests duplicated queue positions; CIRC-2538 ECS pick slips printed from wrong SP; CIRC-2629 extra "Aged to lost" action (ECS); CIRC-2673 declare lost 500
- CIRC-2496 Bug — staff slip preview name always "Transit"
- MODAUD-324 Bug — fee/fine actions missing from Circulation log; MODAUD-323 circ logs not appearing; MODAUD-326 preferred contact changes missing in user version history
- (2026-06-16..07-21) UIREQ-1365 Bug — Requests app crashes opening a request with erased data; CIRC-2497 Bug — staff slip name missing in due date receipt; UICHKOUT-1012 Bug — all custom fields shown at Check out with default Other settings; UICIRC-1354 staff slips localisation; UICIRC-1316 Save button state for "Patron id(s) for checkout scanning"
- (no other Done issues found for UIREQ / UIREQMED / MODREQMED / UICHKOUT / UICIRCLOG in this window)

### TestRail — cases created since 2026-07-21: 3 (3 manual, 0 automated/Karate)

- C1528161 [LoC, API] Check error handling for Secure loan renewal override scenarios that cannot be reproduced on UI — refs UIU-3537,CIRC-2632
- C1525841 [LoC, API] Check error handling for Secure loan renewal scenarios that cannot be reproduced on UI — refs UIU-3537,CIRC-2632
- C1434658 Mediated Requests application is available only in Secure tenant — refs BF-1522
