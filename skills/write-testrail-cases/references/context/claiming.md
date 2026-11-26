---
area: claiming
generated_on: 2026-11-25
generator_version: manual (build-app-context method, refresh)
Release: Umbrellaleaf R2 2026
jira_status: ok
---

# FOLIO Claiming — Business Logic Context

> Built: 2026-11-25 · Release: Umbrellaleaf R2 2026 (ui-claims v3.0.0 in progress at HEAD; v2.0.x = Trillium R1 2026, v1.0.0 = Sunflower R1 2025) · Builder: build-app-context method, manual refresh
> Sources: TestRail suite 21 section 57331 "Claiming" (15 cases; full steps read for C692111, C1322904, C1602437, C1602554, C692167, C692165, C1602553, C1621373, C692233, C648508; C692126, C1322902, C692237, C663254, C1045993 judged by title/refs only); GitHub folio-org/ui-claims (translations/ui-claims/en.json — 55 keys, identical to en_US.json; package.json; src/views/Claiming/*), folio-org/stripes-acq-components (lib/claiming/*, translations/stripes-acq-components/en.json), folio-org/mod-orders (descriptors/ModuleDescriptor-template.json, PiecesClaimingService.java, PieceUpdateFlowManager.java, ExportConfigsCache.java, ErrorCodes.java), folio-org/mod-orders-storage (PieceClaimingRepository.java, views/wrapper_pieces_view.sql, ModuleDescriptor), folio-org/acq-models (claimingCollection.json, pieceStatusBatchCollection.json), folio-org/ui-receiving package.json; Jira UICLAIM (all 25 issues), MODORDERS / MODORDSTOR / UIREC / UIEXPMGR / UIORGS / MODEXPS claiming issues (~50); folio-integration-tests, stripes-testing; docs.folio.org/docs/acquisitions/claiming.
> `[source]` = GitHub source file named in the row; `[cases]` = TestRail cases. Related contexts: `receiving.md` (pieces, statuses, Edit piece form), `organizations.md` (claim integrations), `export-manager.md` (claim jobs), `orders.md` (POL Claiming active / interval).

## What is Claiming

The **Claiming** app lists receiving **pieces** of PO lines with **Claiming active** (default filter: Receiving status = **Late**) across POs and lets staff act on them in bulk: **Send claim** (sets the pieces to Claim sent and creates one claim export job per applicable claim integration, visible in Export manager), **Delay claim**, **Unreceivable**, and **Group by organization**. Claims go out through the organization's **claim integration** (Organizations > Integration details, Integration type Claiming). The app has no detail pane and no settings; pieces are edited in Receiving [ui-claims translations/ui-claims/en.json; ui-claims package.json; C692111; C1322904].

## Record model / data source

- List = `GET /orders/wrapper-pieces` (mod-orders) → `orders-storage/wrapper-pieces` → DB view `claiming_wrapper_pieces_view`: piece + poLine + title + purchaseOrder + `vendorId` (= order vendor), **hard-filtered to `poLine.claimingActive = TRUE`** [mod-orders-storage views/wrapper_pieces_view.sql; UICLAIM-29].
- Organization code/name are resolved separately by organization ids [ui-claims src/utils/api/fetchClaims.ts].
- Page size 50 with Previous/Next pagination [stripes-acq-components RESULT_COUNT_INCREMENT; ui-claims ClaimingList.tsx].

## Piece statuses and transitions

Values: Expected, Received, Late, Claim delayed, Claim sent, Unreceivable [stripes-acq-components lib/constants/piece.js; en.json `piece.pieceStatus.*`].

| From | To | Trigger | Source |
|---|---|---|---|
| Expected | Late | Scheduled job: title `claimingActive` = true AND today (tenant time zone) ≥ piece `receiptDate` + title `claimingInterval` days | PieceClaimingRepository.java; MODORDSTOR-378, MODORDSTOR-433 |
| Claim delayed / Claim sent | Late | Scheduled job: today ≥ piece `statusUpdatedDate` + piece `claimingInterval` days | PieceClaimingRepository.java; MODORDSTOR-374 |
| any (not Late) | Late | Manual "Mark late" in Receiving Edit piece | UIREC-413 |
| Late | Claim sent | Send claim (Claiming or Receiving) | PiecesClaimingService.java |
| Late (any selected) | Claim delayed | Delay claim | useClaimsDelay.js |
| Late (any selected) | Unreceivable | Unreceivable | Claiming.tsx |

Job: `_timer` POST `/orders-storage/claiming/process` — **every 6 hours** (descriptor `unit: hour, delay: 6` at master; it was `delay: 1` in v13.7.0–v13.9.0 after MODORDSTOR-378 moved it from daily to hourly, later raised to 6); testers trigger it manually with an empty-body POST [mod-orders-storage descriptors/ModuleDescriptor-template.json; MODORDSTOR-378; MODORDSTOR-374]. Job updates are audited as Piece Edit events (Status log) by a system user [PieceClaimingRepository.java].

## UI structure

- Search indexes (dropdown): Keyword (default), Title (Receiving titles), Package (POL Package name), Product ID (For the specific title or the package), PO number, POL number, Vendor reference number [en.json `search.index.*`].
- Results pane "Claiming" with record count, Actions menu; first column = row checkbox, header checkbox "Select all claims" [en.json].
- Columns: Expected date, Organization code, Organization name, Title, Display summary, Chronology, Enumeration, POL number, Status [en.json `claiming.results.columns.*`]; column visibility via Actions > column manager (select column excluded) [Claiming.tsx]. Key `claiming.results.columns.piecesToClaim` ("Pieces to claim") exists in en.json but is not used by the list formatter/columns [ui-claims src/views/Claiming/constants.ts].
- Filters (order in UI): Vendor, Order type, Material type (multi-select), Order format, Tags, Location, Receiving status, Acquisition unit, Rush, Received date, Expected receipt date, Receipt due, Created by (Piece), Date created (Piece), Updated by (Piece), Date updated (Piece), Created by (Title), Date created (Title), Updated by (Title), Date updated (Title), Ship to, Custom fields (POL) [ClaimingListFilters.tsx; en.json; C1322904].
- Actions menu: Group by organization, Send claim, Delay claim, Unreceivable [ui-claims en.json `claiming.action.groupByOrganization`; stripes-acq-components en.json `claiming.action.sendClaim` / `delayClaim` / `unreceivable`; C1322904]. The menu label is **"Unreceivable"**; "Mark unreceivable" is only the modal heading.

## API surface

| Endpoint | Module | permissionsRequired | Used for |
|---|---|---|---|
| GET `/orders/wrapper-pieces` | mod-orders (interface wrapper-pieces 1.0) | `orders.wrapper-pieces.collection.get` | List / search / Group by organization |
| POST `/pieces/claim` | mod-orders (pieces.send-claims 1.0) | `pieces.send-claims.collection.post` | Send claim; body `claimingPieceIds` (req), `claimingInterval` (req, days), `internalNote`, `externalNote` |
| PUT `/orders/pieces-batch/status` | mod-orders (pieces 3.1) | `orders.pieces.collection.put` | Delay claim / Unreceivable; body `pieceIds`, `receivingStatus` (req), `claimingInterval`, `internalNote`, `externalNote` |
| POST `/orders-storage/claiming/process` | mod-orders-storage (_timer) | `orders-storage.claiming.process.execute` | Expected/Claim sent/Claim delayed → Late job |
| GET `/data-export-spring/effective-configs?vendorId&acquisitionMethod&type=CLAIMS` | mod-data-export-spring (called by mod-orders) | module permission `data-export.effective-configs.collection.get` | Find claim integrations for a piece |

Sources: mod-orders descriptors/ModuleDescriptor-template.json; mod-orders-storage descriptors/ModuleDescriptor-template.json; acq-models mod-orders/schemas/claimingCollection.json, pieceStatusBatchCollection.json; stripes-acq-components lib/constants/api.js; MODORDERS-1491.

Error codes: `cannotSendClaimsPieceIdsAreEmpty` (400), `cannotFindPiecesWithLatestStatusToProcess` (400), `unableToGenerateClaimsForOrgNoIntegrationDetails` (422, parameters `pieceId`… + `vendorCode`), `userNotAMemberOfTheAcq`, `pieceNotFound` (400, batch status) [mod-orders ErrorCodes.java, PiecesClaimingService.java, PieceUpdateFlowManager.java].

## Permissions

| Okapi permission | Eureka capability set | Gives |
|---|---|---|
| Claims: View claims (`ui-claims.claiming.view`; only sub-permission `module.claims.enabled` → `stripes-core.settings.read`) | `Data - UI-Claims Claiming - View` [stripes-testing capabilitySets.js `uiClaimsClaimingView`; C692165 spells it `data - UI-Claims Claiming - view`] | App visible; **no backend data by itself** [ui-claims package.json] |
| Receiving: View (`ui-receiving.view`) | `Data - UI-Receiving - View` [stripes-testing capabilitySets.js `uiReceivingView`; C692165] | `orders.wrapper-pieces.collection.get` → list + Group by organization [ui-receiving package.json] |
| Receiving: View, edit (`ui-receiving.edit`, also create/delete sets) | `Data - UI-Receiving - Edit` [stripes-testing capabilitySets.js `uiReceivingEdit`; C648508, C1621373] | + `pieces.send-claims.collection.post`, `orders.pieces.collection.put` → Send / Delay / Unreceivable [ui-receiving package.json] |
| Export manager | `Data - UI-Export-Manager Export-Manager - Manage` [stripes-testing capabilitySets.js `uiExportManagerExportManagerManage`; C648508] | Verify claim jobs (C692111, C1602437, C648508) |

Actions are hidden (not just disabled) when the permission is missing: `IfPermission` wraps each item [Claiming.tsx; C692167]. The TestRail cases spell capability sets in lower case (`data - UI-… - view`); the Cypress dictionary builds the same names from type `DATA` + resource + action. docs.folio.org lists "Claims: View claiming" → capability `orders-storage.claiming.all` — differs from package.json (see Known Gaps).

## Key Business Rules for Test Cases

### List and search
1. Only pieces of PO lines with **Claiming active** are ever listed; the view hard-filters `poLine.claimingActive = TRUE`, so unchecking Late never shows non-claiming POL pieces [wrapper_pieces_view.sql; UICLAIM-29; C1322904].
2. Default filter on open = Receiving status **Late**; the user can change it (e.g. Claim delayed, Unreceivable) [Claiming.tsx DEFAULT_FILTERS; C1322904].
3. The `claimingActive` ("POL claim active") filter was removed (UICLAIM-29, ui-claims 2.0.2) — do not assert it.
4. "Reset all" clears all filters including Late; then no search runs and the empty-state text "Choose a filter or enter a search query to show results." is shown [C1322904; Claiming.tsx breakWithDefaults].
5. Default sort: Expected date ascending (oldest first); sortable headers: Expected date, Status, Display summary, Chronology, Enumeration, Title; Organization code/name and POL number headers are not sortable [search.ts; constants.ts; UICLAIM-1].
6. Group by organization sorts by the order's vendor id (groups pieces per organization); selecting it again when already grouped has no effect [Claiming.tsx sortByOrg; UICLAIM-2; C1322904].
7. PO number and POL number searches are exact (`==`, wildcards allowed); other indexes are fuzzy; Keyword ORs all six indexes [search.ts; UICLAIM-31].
8. Changing the search index does not run a search; search runs only on Search/Enter (UICLAIM-32, ui-claims 3.0.0).
9. Material type filter is multi-select and, combined with Order format, matches the format-specific material type (P/E mix checks both eresource and physical) [search.ts; UICLAIM-25].
10. Location filter matches POL locations or `searchLocationIds` [search.ts].
11. Pieces without "Expected receipt date" never become Late via the job (still can be marked Late manually) [folio-integration-tests piece-status-transitions-claiming.feature; UIREC-413].
12. An acquisition unit assigned to the **organization** does not hide or block its pieces; Vendor shows "-" where the org is restricted; the restricted org is not found in the Vendor filter Organization look-up, other orgs' pieces stay listed and claimable, and Send claim still creates one export job per integration (Export job details "Organization" = "-") [UICLAIM-36; C1621373].

### Actions — common
13. Send claim, Delay claim and Unreceivable are disabled until at least one piece is checked; Group by organization is always enabled [Claiming.tsx; UICLAIM-3/4/5; C1322904].
14. Cancel in any action modal closes it, changes nothing and keeps checkboxes checked [UICLAIM-3/4/5; C692111; C1322904].
15. After a successful action selections are cleared and the list is refetched; with default Late filter the acted pieces disappear from the list [Claiming.tsx; C1322904].
16. On error the modal closes, list is refetched and the error toast is shown; for generic errors (e.g. acq-unit restriction) the checkboxes stay checked [Claiming.tsx; C1602554].
17. Acquisition units on the **title** are enforced: a user not in the title's AU gets "Failure on delay claims." / "Failure on mark unreceivable" / "Failure on send claims." and no piece changes; a mixed selection fails entirely [MODORDERS-1495 (mod-orders 13.1.7); C1602554].
18. Claiming actions never clear existing piece Internal/External notes; Delay and Unreceivable send no notes [MODORDERS-1492 (13.1.7); C1602437].

### Send claim
19. Modal "Send claim(s)" contains the text "Claim <count> piece(s)?", required "Claim expiry date" (only future dates; past/today → "Selected date must be later than the current date"), "Internal note", "External note", "Cancel", "Save & close" [stripes-acq-components SendClaimsModal.js, FieldClaimingDate utils.js; C692111].
20. `claimingInterval` sent = days from today to the chosen date; the piece becomes Late again when statusUpdatedDate + interval passes [getClaimingIntervalFromDate.js; PieceClaimingRepository.java].
21. Success: toast "Claims are currently being processed. Please check Export manager for job status."; pieces → Claim sent with interval and notes [en.json; PiecesClaimingService.java; C692111].
22. Only Late pieces are claimed; non-Late pieces in the selection are silently skipped; if none is Late → 400 and toast "Cannot find pieces with "Late" status to process." [PiecesClaimingService.java; MODORDERS-1491].
23. Claim integrations are resolved per (vendor, POL acquisition method) via effective configs; one export job is created per applicable integration, containing only its pieces; a piece matching several integrations goes into each job [MODORDERS-1491; claims-export-acquisition-method.feature; C692126, C648508, C1322902].
24. A claim integration with acquisition methods takes only those methods; one without methods takes methods no other claim integration of the vendor selects [claims-export-acquisition-method.feature].
25. If **any** selected Late piece has no applicable integration, the whole request is rejected (422) before any update: no status changes, no jobs, toast "Unable to generate claims for <b>{orgCodes}</b> because no claim integrations exist." with the first such vendor code only; all selected pieces stay checked [PiecesClaimingService.java; MODORDERS-1491; C692111].
26. Pieces whose order vendor organization is not a vendor (`isVendor` false) are dropped from claiming [PiecesClaimingService.java createClaimPiece].
27. Each claim run creates new jobs; previous claim jobs stay in Export manager (Organizations toggle, Integration type "Claims") [MODORDERS-1264; C692237].
28. CSV claim file name `csv_claims_<Vendor code>_<Export method>_YYYY-MM-DD_HH-MM-SS.csv`; columns include POL number, Vendor order number, Account number, Expected date, Title, Display summary, Chronology, Enumeration, Quantity, External note [C692111; MODEXPW-584].

### Delay claim
29. Modal "Delay claim(s)" with required "Delay to" date (future only), "Cancel", "Save & close"; success toast "Claims are successfully delayed."; status → Claim delayed with interval = days until "Delay to" [DelayClaimsModal.js; useClaimsDelay.js; en.json; C1322904].
30. Re-delaying a piece already in Claim delayed saves the new interval and resets statusUpdatedDate (new Status log record) so Late happens on the new date; each delay adds a Status log row ("Claim delayed" + new interval; earlier rows stay), including from Receiving Edit piece (toast "The piece was successfully saved") [MODORDERS-1494 (13.1.7); C1602553].
31. Delay claim is not restricted to Late pieces (any selected piece is updated) — Late check exists only for Send claim [useClaimsDelay.js; PieceUpdateFlowManager.java].

### Unreceivable
32. Modal heading "Mark unreceivable", text "Mark <count> piece(s) unreceivable?", buttons "Cancel" / "Save & close"; success toast "Claims are successfully marked unreceivable"; piece shown in Receiving "Unreceivable" accordion [MarkUnreceivableModal.tsx; en.json; UICLAIM-5; C1322904].
33. POL receipt status is recalculated only for **One-time Open** orders: Unreceivable can move it to Partially/Fully received; Claim sent/Claim delayed keep "Awaiting receipt"; Ongoing or Cancelled POLs are skipped [PieceUpdateFlowManager.java updatePoLineReceiptStatus; MODORDERS-1027; MODORDERS-1030].
34. Unreceivable does not change the linked item status (stays On order) — enhancement MODORDERS-1437 is Open; do not assert Unavailable.

### Permissions, ECS, organizations
35. Without Receiving view no pieces load; without Receiving edit only "Group by organization" is shown in Actions (C692165, C692167; UICLAIM-8).
36. ECS: for a central-tenant POL with a member-tenant location, the piece is NOT listed in the Claiming app of the member tenant; it is claimed from Receiving (Edit piece > Send claim, popup text "This will generate a claim job which can be viewed in Export manager. Continue?") and the job appears in Export manager in the Central tenant; wrapper-pieces results are limited to the tenants the user is affiliated with (central ordering) [WrapperPieceStorageService.java; C692233].
37. Claim integration = Organization integration with Integration type "Claiming", Transmission method FTP or File download, File format CSV or EDI; no Scheduling accordion for Claiming [UIORGS-442; ui-organizations en.json `integration.info.integrationType.claiming`].
38. Claim file formats: CSV (name above) or EDI (`edi_claims_<Vendor code>_<Export method>_YYYY-MM-DD_HH-MM-SS`; segments UNB/UNH/NAD+BY/NAD+SU/DOC+220/LIN/PIA/IMD+L+080 (Display summary:Chronology:Enumeration)/IMD+L+109 (Publisher)/STS+UP1::9+CSD::9+55/RFF+LI/RFF+SNA/RFF+ACT:::1); CUX, PRI+AAF, PRI+AAB, RFF+BFN and LOC+7 segments are NOT present; FTP integration also writes the file to the Order directory [C648508; MODEXPW-584].

## Exact UI Texts

> From `folio-org/ui-claims` `translations/ui-claims/en.json` and `folio-org/stripes-acq-components` `translations/stripes-acq-components/en.json` at HEAD (2026-11-24; en_US.json identical for these keys). `{x}` are placeholders; `<b>` renders bold. Use verbatim.

### Toast Messages

| Key | Text | Source |
|---|---|---|
| `ui-claims.claiming.delayClaim.success.message` | Claims are successfully delayed. | [source] [cases C1322904] |
| `ui-claims.claiming.delayClaim.error.message` | Failure on delay claims. | [source] [cases C1602554] |
| `ui-claims.claiming.sendClaim.success.message` | Claims are currently being processed. Please check Export manager for job status. | [source] [cases C692111] |
| `ui-claims.claiming.sendClaim.error.message` | Failure on send claims. | [source] [cases C1602554] |
| `ui-claims.claiming.markUnreceivable.success.message` | Claims are successfully marked unreceivable | [source] [cases C1322904] |
| `ui-claims.claiming.markUnreceivable.error.message` | Failure on mark unreceivable | [source] [cases C1602554] |
| `ui-claims.error.code.cannotFindPiecesWithLatestStatusToProcess` | Cannot find pieces with "Late" status to process. | [source] |
| `ui-claims.error.code.unableToGenerateClaimsForOrgNoIntegrationDetails` | Unable to generate claims for <b>{orgCodes}</b> because no claim integrations exist. | [source] [cases C692111] |

### Modals

| Key | Text | Source |
|---|---|---|
| `stripes-acq-components.claiming.modal.sendClaim.heading` | {count, select, undefined {Send claim} other {Send {count, plural, one {claim} other {claims}}}} | [source] |
| `ui-claims.claiming.sendClaim.modal.message` | Claim <b>{count, number}</b> {count, plural, one {piece} other {pieces}}? | [source] [cases C692111] |
| `stripes-acq-components.claiming.modal.sendClaim.field.claimExpiryDate` | Claim expiry date | [source] |
| `stripes-acq-components.claiming.modal.sendClaim.field.internalNote` / `.externalNote` | Internal note / External note | [source] |
| `stripes-acq-components.claiming.modal.delayClaim.heading` | {count, select, undefined {Delay claim} other {Delay {count, plural, one {claim} other {claims}}}} | [source] |
| `stripes-acq-components.claiming.modal.delayClaim.field.delayTo` | Delay to | [source] |
| `ui-claims.claiming.markUnreceivable.modal.heading` | Mark unreceivable | [source] |
| `ui-claims.claiming.markUnreceivable.modal.message` | Mark <b>{count, number}</b> {count, plural, one {piece} other {pieces}} unreceivable? | [source] [cases C1322904] |
| `stripes-acq-components.FormFooter.cancel` / `.save` | Cancel / Save & close | [source] |
| `stripes-acq-components.validation.dateAfter` | Selected date must be later than the current date | [source] |
| `stripes-acq-components.validation.required` | Required! | [source] |

### Actions and statuses

| Key | Text | Source |
|---|---|---|
| `ui-claims.claiming.action.groupByOrganization` | Group by organization | [source] |
| `stripes-acq-components.claiming.action.sendClaim` | Send claim | [source] |
| `stripes-acq-components.claiming.action.delayClaim` | Delay claim | [source] |
| `stripes-acq-components.claiming.action.unreceivable` | Unreceivable | [source] [cases C1322904] |
| `stripes-acq-components.piece.pieceStatus.*` | Expected, Received, Late, Claim delayed, Claim sent, Unreceivable | [source] |
| `ui-claims.permission.claiming.view` | Claims: View claims | [source] |

## Existing coverage

### TestRail (section 57331)
C692111 (no integration + one integration, search indexes, CSV file), C692126 (two integrations), C648508 (two orgs, 1 and 2 integrations), C1322902 (two orgs, one integration each), C692165 / C692167 (capabilities), C663254 (keyboard shortcuts), C692237 (consecutive runs in Export manager), C692233 (ECS), C1045993 (localization), C1322904 (Unreceivable, Delay, Group by, filters), C1602437 (notes kept, MODORDERS-1492), C1602553 (re-delay, MODORDERS-1494), C1602554 (title AU restrictions, MODORDERS-1495), C1621373 (organization AU, UICLAIM-36).
Not covered yet: acquisition-method routing of claim integrations (MODORDERS-1491), Send claim with mixed Late/non-Late selection, Delay/Unreceivable of non-Late pieces, column manager, sorting by each column, Received date / Receipt due / Ship to / Custom fields filters, vendor `isVendor` = false.

### Automation
- Karate (folio-integration-tests, `acquisitions/src/main/resources/thunderjet/`): `mod-orders/features/piece-status-transitions-claiming.feature`, `piece-batch-job.feature`, `pieces-batch-update-status.feature`, `poline-claiming-interval-checks.feature`, reusables `mod-orders/reusable/claim-pieces.feature`, `update-pieces-batch-status.feature`; `mod-data-export-spring/features/claims-export.feature` (CSV/EDI to FTP/SFTP, no-config case, 500 pieces), `claims-export-acquisition-method.feature`.
- Cypress (stripes-testing): `cypress/e2e/claiming/*.cy.js` — C692237, C692126, C1322902, C692167, C692165; fragment `cypress/support/fragments/claiming/claiming.js`; receiving claim specs in `cypress/e2e/receiving/*claim*.cy.js`; modals `cypress/support/fragments/receiving/modals/sendClaimModal.js`, `delayClaimModal.js`.

## Known Gaps

- **Partial claiming conflict:** UICLAIM-3 test table and UIEXPMGR-114 expect jobs for organizations with integrations (or a default CSV/download claim) when another selected organization has none; current mod-orders rejects the whole request (MODORDERS-1491 "unchanged behaviour", C692111 step 13). Follow BE/C692111.
- UI error message says "no claim integrations exist" while BE message now says "no claims integration applies to the claimed pieces" (MODORDERS-1491) — UI uses its own translation, so assert the UI text.
- Whether the Organizations UI exposes acquisition methods for **Claiming** integrations (needed for MODORDERS-1491 routing) not confirmed in ui-organizations; only `integration.edi.defaultAcquisitionMethods` "Automate order export for acquisition methods" found.
- UICLAIM-34 (In Refinement): informative toast for `userNotAMemberOfTheAcq` not implemented; generic "Failure on …" toasts shown today.
- UICLAIM-35 (Open): pieces of AU-restricted titles still listed; hiding them is not implemented.
- MODORDERS-1464 (In Refinement): "No acquisition unit" filter returns no records.
- MODORDERS-1495 and MODORDERS-1492 are "In bugfix review" in Jira but released in mod-orders 13.1.7 and verified on Trillium Bugfest.
- UIREC-518 (same-interval re-delay from Receiving Edit piece) and UIREC-519 (rejected Send claim from Receiving still saves date/notes) are open Receiving defects.
- docs.folio.org Claiming page is outdated: still lists "POL claim active (default)" filter and says unchecking it shows non-claim-active pieces; permission table maps to `orders-storage.claiming.all` instead of package.json sets.
- Timer frequency: resolved from the module descriptor — MODORDSTOR-378 (13.7.0) set hourly (`delay: 1`, still in v13.9.0); master descriptor now has `delay: 6` (6 hours). The commit/Jira that changed it to 6 was not identified, and the effective schedule on environments (tenant override of `_timer`) is not verified.
- Capability set names are confirmed only for Claiming view, Receiving view/edit and Export-Manager manage (Cypress dictionary); `UI-Orders Orders - view` and the ECS `settings - UI-Consortia-Settings Settings NetworkOrdering - view` seen in C1602553/C692233 preconditions were not checked in the dictionary.
- `Pieces to claim` column (UICLAIM-1) has a translation key but no column; Organization code/name sorting not supported.
