# FOLIO Receiving App — Business Logic Context

> Built: 2026-11-24 (refresh; previous build 2026-06-17, delta 2026-11-07) · Release: Umbrellaleaf R2 2026 (master: ui-receiving 9.0.0 IN PROGRESS; Trillium R1 2026 = ui-receiving 8.0.x / mod-orders 13.1.x) · Builder: build-app-context (refresh mode, full source pass)

> Sources:
> - GitHub folio-org/ui-receiving master: `translations/ui-receiving/en.json` (318 lines; `en_US.json` identical except it lacks `numberGenerator.noPermission`), `package.json` (stripes.permissionSets), `CHANGELOG.md`, `src/Piece/PieceForm/PieceFormActionButtons/constants.js`
> - GitHub folio-org/stripes-acq-components master: `translations/stripes-acq-components/en.json` (piece statuses, claiming actions, shared Delete Holdings modal, routing lists)
> - GitHub folio-org/mod-orders master: `descriptors/ModuleDescriptor-template.json`, `src/main/java/org/folio/rest/core/exceptions/ErrorCodes.java`; folio-org/mod-orders-storage `descriptors/ModuleDescriptor-template.json` (claiming timer); folio-org/acq-models `mod-orders-storage/schemas/piece.json`
> - Jira (anonymous REST): UIREC Done/Closed stories+bugs resolved ≥ 2024-06-01 (75 issues: Ramsons → Umbrellaleaf), MODORDERS receiving-related Done/Closed ≥ 2024-06-01 (~75 issues); ~35 read in detail with comments
> - folio-integration-tests `acquisitions/src/main/resources/thunderjet/mod-orders/` (Karate); stripes-testing `cypress/e2e/receiving` (113 specs), `cypress/e2e/orders/receiving-and-check-in` (16), `cypress/e2e/claiming` (5), `cypress/e2e/consortia/receiving` (1)
> - docs.folio.org source `content/en/docs/Acquisitions/Receiving/_index.md`; Confluence "Receiving - Trillium - Notes on functionality"
> - TestRail section subtree 17995 (suite 21: sections 17995, 17996, 36735; 244 cases at 2026-06-17 build) — **not re-queried this refresh** (bridge disconnected); case IDs below come from the earlier build, Jira comments and stripes-testing spec titles.
> Tags: [confirmed] = translation key + case/spec; [from source] = translation/code only; [from cases] = TestRail/spec wording only.

---

## What is Receiving

The Receiving app confirms that materials ordered through the Orders app have been physically delivered to the library. It manages receiving titles and their pieces (individual receiving units) tied to a Purchase Order Line (POL), plus claiming (Late / Claim sent / Claim delayed), binding, routing lists and CSV export. Receiving is possible only for Open or Closed orders (docs `_index.md` → Order status; Closed includes Cancelled orders — UIREC-497, C430249).

---

## Key Terms

| Term | Definition |
|---|---|
| Receiving title | The title of ordered material; can have one or more pieces. Created automatically for non-package POLs; for package POLs added from Orders ("Package titles") or Receiving ("New title") (docs) |
| Piece | A single receivable unit under a receiving title; required fields `format`, `poLineId`, `titleId`, `receivingStatus` (acq-models `piece.json`) |
| Package | A POL representing multiple titles; titles must be added before receive |
| Receiving workflow | POL setting: `Synchronized order and receipt quantity` (pieces auto-created on open; adding/removing pieces changes POL quantity) or `Independent order and receipt quantity` (no auto pieces) (en.json `title.receivingWorkflow.*`, docs) |
| Receive / Quick receive | Confirm piece delivery (batch full-screen or single piece from Edit piece) |
| Unreceive | Move received piece back to Expected |
| Unreceivable | Mark a piece that will never arrive; reversible with `Expect` |
| Sequence | Piece display order, `[n] of N`; auto-assigned, editable (UIREC-465/466) |
| Bound item | New Inventory item created by Bind pieces from several received pieces |

---

## Piece Statuses and Transitions

Status values (stripes-acq-components `piece.pieceStatus.*`): `Expected`, `Received`, `Late`, `Claim delayed`, `Claim sent`, `Unreceivable`. `Late`, `Claim delayed`, `Claim sent` pieces stay in the **Expected** accordion; `Received` → Received accordion; `Unreceivable` → Unreceivable accordion (docs → Claiming; C844841).

| From | Action / trigger | To | Source |
|---|---|---|---|
| Expected / Late / Claim sent / Claim delayed | Quick receive, Receive (full screen) | Received | PieceFormActionButtons/constants.js |
| Expected / Claim sent / Claim delayed | `Mark late` | Late | UIREC-413, C656338 |
| Expected-group | `Send claim` (Claim expiry date, optional internal/external note) | Claim sent | UIREC-412, C692110, C423991 |
| Expected-group | `Delay claim` (`Delay to` date) | Claim delayed | stripes-acq-components `claiming.action.delayClaim`, C423985 |
| Expected-group | `Unreceivable` | Unreceivable | C436833, C436836 |
| Expected (claiming active, receipt date + interval passed) | claiming batch job | Late | C436495, Karate `piece-status-transitions-claiming.feature` |
| Claim sent (expiry date passed) / Claim delayed (delay date passed) | claiming batch job | Late (see Known Gaps re docs) | C436793, C436794, C436736 |
| Received | `Unreceive` | Expected | en.json `piece.actions.unreceive.*` |
| Unreceivable | `Expect` | Expected | C423617, C423668, C430229 |

- Claiming batch job: `POST /orders-storage/claiming/process`, run by a `_timer` every **6 hours** (mod-orders-storage ModuleDescriptor) — a date passing does not flip the status until the job runs.
- Status changes are logged in the `Status log` accordion (columns `Status change`, `Date`, `Interval`, `Updated by`; Interval only for Send/Delay claim) (en.json `piece.statusChangeLog.*`, docs).

---

## Receiving Workflow

### Quick receive
1. Open piece from Expected accordion (Edit piece full-screen form, UIREC-368).
2. Dropdown next to `Save & close` → `Quick receive`.
3. Piece moves to Received.

### Receive (full screen, batch)
1. Expected accordion → Actions → `Receive`.
2. If the title has **more than 50** expected pieces, a `Select expected date range` modal (Start/End date, both required) appears first; ≤ 50 pieces opens the screen directly (UIREC-461, C813585).
3. Select pieces, edit per-row fields, click `Receive` (or `Receive & load more` per 50-piece batch).

### Must acknowledge receiving note
If the POL has `Must acknowledge receiving note`, the `Receiving note` dialog appears on receive; user must click `Continue` (en.json `title.isAcknowledged`, `piece.receivingNoteModal.title`, `piece.actions.confirm`; docs).

### Receive side effects
1. Piece status → Received; `Received date` set.
2. Connected item with status `On order` or `Order closed` → `In process`; items in other statuses keep their status (en.json `piece.itemStatus.info`; C367971).
3. POL receipt status recalculates (Partially Received / Fully Received; ongoing orders show `Ongoing`).
4. PO can auto-close when all POLs are fully received and fully paid.
5. Receiving against a Closed (incl. Cancelled) order shows the `Order closed` confirmation first — also for Quick receive since 8.0.2 (UIREC-497).

### Unreceive side effects
1. Piece returns to Expected.
2. Associated item is set to `On order` (en.json `piece.actions.unreceive.message`); the item is **not** deleted.
3. Piece keeps Barcode, Call number, Accession number (UIREC-483, fixed 8.0.0); `sequenceNumber`/`displayToPublic` are sent in the unreceive payload (UIREC-474).
4. POL receipt status recalculates.

---

## Inventory Integration

| Event | Item effect | Source |
|---|---|---|
| Add piece with `Create item` checked (POL Create inventory = Instance, holdings, item) | Item created on save (status `On order`; `Order closed` if the POL is cancelled) | docs → Piece details; MODORDERS-1140 |
| Receive | `On order`/`Order closed` → `In process`; other statuses unchanged | en.json `piece.itemStatus.info`; C367971 |
| Unreceive | Item set to `On order` | en.json `piece.actions.unreceive.message` |
| Mark Unreceivable / add piece on cancelled POL | Item `Order closed` | MODORDERS-1140 (via MODORDERS-1173) |
| Delete expected piece | Connected item deleted | docs → Deleting an expected piece |
| Delete received piece | Item kept | docs → Deleting a received piece |
| Edit received piece | Only edited fields are written to the item; untouched fields are not overwritten | MODORDERS-1330/1331, C432301-C432304 |
| Edit piece of a suppressed item | Item stays suppressed (suppress flag no longer synced) | MODORDERS-1316, C784497 |

Check in is a separate workflow: after receive, Check in moves the item from `In process` to `Available` (or `Awaiting pickup` if a request exists) (docs → Receiving a piece).

---

## Package POL Rules

- Package POL cannot be received directly; titles must be added first (Orders `Package titles` or Receiving `New title`).
- A newly created title without pieces appears in the POL's `Package titles` (C466283, C466284; ECS central C468239).
- `Remove from package` (title Actions / title edit form) deletes the title's pieces and related Holdings/Items; holdings with items of other orders are kept (en.json `title.confirmationModal.removeFromPackage.message`; UIREC-448; Karate `unlink-title.feature`). Hidden for non-package titles (UIREC-452) and titles from "foreign" tenants (UIREC-454). Blocked when the title has Received pieces (ErrorCodes `existingReceivedPiecesTitleRemoval`).
- A non-package POL may hold only one title (en.json `errors.multipleNonPackageTitles`).

---

## API Surface (mod-orders / mod-orders-storage)

| Method + path | permissionsRequired |
|---|---|
| `POST /orders/receive` · `POST /orders/check-in` · `POST /orders/expect` | `orders.receiving.collection.post` · `orders.check-in.collection.post` · `orders.expect.collection.post` |
| `GET /orders/receiving-history` | `orders.receiving-history.collection.get` |
| `GET/POST /orders/pieces`, `GET/PUT/DELETE /orders/pieces/{id}` | `orders.pieces.collection.get`, `orders.pieces.item.post/get/put/delete` |
| `POST /orders/pieces-batch` · `PUT /orders/pieces-batch/status` | `orders.pieces.collection.post` · `orders.pieces.collection.put` |
| `GET /orders/pieces-requests` | `orders.piece-requests.collection.get` |
| `GET /orders/wrapper-pieces[/{id}]` | `orders.wrapper-pieces.collection.get` / `.item.get` |
| `GET/POST /orders/titles`, `GET/PUT/DELETE /orders/titles/{id}` | `orders.titles.*` (DELETE = unlink from package, `deleteHoldings` param) |
| `POST /orders/holding-detail` (decides Delete/Keep Holdings modal) | `orders.holding-detail.item.post` (MODORDERS-1371) |
| `POST /orders/bind-pieces` · `DELETE /orders/bind-pieces/{id}` | `orders.bind-pieces.collection.post` · `orders.bind-pieces.item.delete` |
| `POST /pieces/claim` | `pieces.send-claims.collection.post` |
| `GET/POST/PUT/DELETE /orders/routing-lists[/{id}]`, `GET /orders/routing-lists/{id}/template` | `orders.routing-lists.*`, `orders.routing-lists-template.item.get` |
| `POST /orders-storage/claiming/process` (timer, 6 h) | `orders-storage.claiming.process.execute` |
| `GET /orders-storage/titles/{id}/sequence-numbers` | `orders-storage.titles.sequence-number.get` |

---

## Main Sections / UI Structure

### Panes / pages
- `Receiving` (`meta.title`), `Search & filter`, search results, title details pane, `New title` / `Add title` [from source]
- Edit piece / Add piece full-screen form (`piece.pieceForm.edit.title` = `Edit piece`, `piece.pieceForm.create.title` = `Add piece`) with accordions `Piece details`, `Item details` (or `Item details (original)` for bound pieces), `Status log` [confirmed]
- Full-screen Receive view (subheader `{count} of {totalRecords} receiving pieces`), Bind pieces page, Routing list page [from source]
- Connected Tasks/Jobs pane on title and piece pages, opened from the piece form header icon (UIREC-511; C1554507-C1554509)

### Title details accordions
`Title information`, `POL details`, `Expected`, `Received`, `Unreceivable`, `Bound items`, `Routing lists` (Routing lists only for Physical / P/E Mix; docs) [confirmed — en.json `title.*`, `routing.list.label`]

### Piece lists
- Columns include left-most sortable `Sequence`; Expected sorted ascending, Received/Unreceivable descending (UIREC-465); `Holdings location`, `Display to public` columns (C844842).
- Received accordion excludes Bound pieces by default; `Bound`/`Not bound` filter in its Actions menu (UIREC-264; kept by design — UIREC-449).
- Main results list: Actions → show/hide columns, persists for the session (UIREC-494); `Vendor` column, not sortable (UIREC-499); full-screen Receive table has its own column manager (UIREC-492).

---

## Search and Filters

Search options (en.json `search.*`): `Keyword`, `Title (Receiving titles)`, `Package (POL Package name)`, `Product ID (For the specific title or the package)`, `PO number`, `POL number`, `Vendor reference number`, `Barcode`.

Filters (en.json `filter.*`): `Order status`, `Vendor`, `Order type`, `Material type`, `Order format`, `Location`, `Receiving status`, `Rush`, `Bindery active`, `Supplements`/`Non supplements`, `Created by`, `Updated by`, `Date created`, `Date updated`, and piece-level `Created by (Piece)`, `Updated by (Piece)`, `Date created (Piece)`, `Date updated (Piece)`; docs also list Tags, Acquisition unit, Received date, Expected receipt date, Receipt due.
- Location filter matches PO line locations **or** holdings (UIREC-358).
- Titles with special characters are found via `Title (Receiving titles)` (UIREC-500, fixed 8.0.3).
- Pending orders are visible but not receivable.

---

## Exact UI Texts

> Use verbatim. `<b>…</b>` renders bold; `{x}` interpolates. Keys are ui-receiving `en.json` unless prefixed `acq:` (stripes-acq-components en.json).

### Toast messages
| Event | Text | Key / source |
|---|---|---|
| Piece saved | `The piece was successfully <b>saved</b>` | `piece.actions.savePiece.success` [confirmed] |
| Piece save failed (incl. budget/encumbrance block) | `The piece was <b>not saved</b>` | `piece.actions.savePiece.error`; C1250459, C1250460 |
| Title receive / unreceive | `Receiving successful` / `Unreceiving successful` | `title.actions.receive.success` / `unreceive.success` [confirmed] |
| Piece quick received | `The piece <b>{enumeration}</b> was successfully <b>received</b>` | `piece.actions.checkInItem.success` [confirmed] |
| Piece unreceived | `The piece <b>{enumeration}</b> was successfully <b>unreceived</b>` | `piece.actions.unreceivePiece.success` [confirmed] |
| Piece deleted / not deleted | `The piece <b>{enumeration}</b> was successfully <b>deleted</b>` / `The piece <b>{enumeration}</b> was not deleted` | `piece.actions.delete.success` / `.error` |
| Expect | `Pieces expect successful` / `Pieces expect failed` | `title.actions.expect.*` |
| Title added | `The title <b>{title}</b> has been successfully added for <b>PO line {poLineNumber}</b>` | `title.actions.save.success`; C423486 |
| Sequence changed | `The sequence of the piece was successfully changed from <b>{oldSequenceNumber}</b> to <b>{newSequenceNumber}</b>` | `piece.actions.savePiece.changeSequenceNumber.success`; UIREC-466 |
| Bind | `Item <b>{barcode}</b> created successfully` / `Item created successfully` (no barcode) | `bind.pieces.create.success.withBarcode` / `.noBarcode` |
| Remove from package | `Related pieces and any related inventory Holdings and Items have been deleted.` | `title.confirmationModal.removeFromPackage.success` |
| Export started | `Export has been started successfully` | `exportSettings.success` |
| Routing list | `The Routing list was successfully created.` / `updated.` / `deleted.` | acq: `routing.list.*.success` |

### Edit piece form — `Save & close` dropdown by piece status (current master)

Source: `src/Piece/PieceForm/PieceFormActionButtons/constants.js`; C844841.
- **Expected / Claim sent / Claim delayed**: `Save and create another`, `Quick receive`, `Mark late`, `Send claim`, `Delay claim`, `Unreceivable`, `Delete`
- **Late**: same without `Mark late` (UIREC-413, C656338)
- **Received**: `Save and create another`, `Unreceive`, `Delete`
- **Unreceivable**: `Save and create another`, `Expect`, `Delete`; holding/affiliation not editable (UIREC-361, C844841)
- **Add piece (new)**: no `Send claim` (UIREC-444) and no `Delete`
- Labels: `piece.action.button.*` (Save and create another, Quick receive, Unreceive, Delete, Expect, Mark late); acq: `claiming.action.sendClaim` = `Send claim`, `claiming.action.delayClaim` = `Delay claim`, `claiming.action.unreceivable` = `Unreceivable`.
- The dropdown is disabled only when all its actions are disabled (UIREC-418, C594368).

### Title information accordion — claiming fields (C423486)
`Claiming active` checkbox and `Claiming interval` field (en.json `title.claimingActive`, `title.claimingInterval`). Interval is inactive while `Claiming active` is unchecked; default inherited from the vendor organization's `Claiming interval`; shows `-` when claiming is off. Changing the POL claiming interval updates the (non-package) title (C770417; Karate `poline-claiming-interval-checks.feature`).

### Confirm dialogs and prompts
| Trigger | Text | Key |
|---|---|---|
| Unreceive confirm | `Unreceiving piece` / `Are you sure you want to unreceive piece? It will return to a status of Expected and any associated item will be set as On-Order.` | `piece.actions.unreceive.heading` / `.message` |
| Delete piece | `Delete piece` / `Are you sure you want to delete piece?` | `piece.delete.heading` / `.message` |
| Delete piece that leaves holding empty | `This piece is connected to records in inventory. There are no other items connected to the related Holdings record. After deleting this piece would you like FOLIO to delete the Holdings?`; buttons `Delete Holdings`, `Delete item`, `Delete Holdings and Item` | `piece.delete.deleteHoldingsAndItem.message`, `piece.actions.delete.*` |
| Holding change abandons holding (shared modal) | heading `Delete Holdings`; `This piece is connected to records in inventory. After this edit there will be no other purchase order lines, pieces OR items connected to the related Holdings record(s). After making this change would you like FOLIO to delete the Holdings record(s)?`; `Keep Holdings` | acq: `holdings.deleteModal.*` (see Known Gaps vs C503244 wording) |
| Receive against closed order | `Order closed` / `The order linked to this Title is closed. Are you sure you want to receive this piece(s)?` | `piece.confirmReceiving.*` |
| Send claim, integration exists | `This will generate a claim job which can be viewed in Export manager. Continue?` | `piece.sendClaim.withIntegration.message` |
| Send claim, no integration | `No claiming integration exists for this organization. Piece will be set to "Claim sent" status. Continue?` | `piece.sendClaim.withoutIntegration.message` |
| Send claim / Delay claim modals | `Send claim` (`Claim expiry date`, `Internal note`, `External note`); `Delay claim` (`Delay to`) | acq: `claiming.modal.*` |
| Remove from package | `Remove from package` / `Removing this title from the package will delete all related pieces and delete any related inventory Holdings and Items. Holdings that contain items that are not related to this order and receiving title will not be deleted.` | `title.paneTitle.removeFromPackage`, `title.confirmationModal.removeFromPackage.message` |
| Bind with requests | `There are requests on the following pieces: <b>{barcodes}</b>. <b>Transfer requests</b> to the new record?` — `Transfer` / `Do not transfer` | `bind.pieces.modal.transfer.message`, `.button.*` |
| Receive item with open request | `An item has an open request` / `Multiple items have open requests` | `requests.title.*` |
| Synchronized title warning | `Adding or removing pieces from this record will update the purchase order line quantity and location detail and in turn the estimated price.` | `title.changePieceQuantityMessage` |
| Export | `Export settings` modal; `This export could take a few minutes. …` | `exportSettings.*` |

### Error / validation messages
| Condition | Text | Key |
|---|---|---|
| Title exists | `The title {title} for PO line {poLineNumber} already exists` | `title.actions.save.error.titleExist` |
| Piece already received | `The piece record is already received` | `errors.pieceAlreadyReceived` |
| Barcode not unique | `Barcode must be unique, piece and item data could not be updated.` | `errors.barcodeIsNotUnique` |
| Create item not allowed | `Create item for piece format is not allowed. Please check "create inventory" value in the purchase order line` | `errors.createItemForPieceIsNotAllowedError` |
| Piece on pending order | `Creating piece for pending order is not possible. Please open order` | `errors.createPiecePendingOrderError` |
| Delete last synchronized piece | `The piece was not deleted because you cannot delete all pieces when ordering and receiving quantity are synchronized.` | `errors.lastPiece`; UIREC-481 |
| Acq-unit restriction | `Action is restricted by acquisition unit. User is not assigned to the specified acquisitions unit.` | `errors.userNotAMemberOfTheAcq` |
| Fund distribution in amounts | `Pieces can not be added to or deleted from this Title until all the Fund distributions on the related purchase order line are converted from amounts to percentages.` | `errors.receivingProcessEncumbrancesError` |
| Fund not visible/deleted | `One or more funds could not be loaded. The fund(s) may have been deleted or you may not have permission to view the associated fund(s).` | `errors.fundsNotFound`; UIREC-488 |
| POL number too long | `POL number is invalid or bigger than {count, number} alphanumeric characters.` | `errors.polNumberInvalidOrTooLong` |
| Item update failed on receive | `Piece <b>{enumeration}</b> was not received. The item record failed to be updated` | `errors.itemUpdateFailed` |
| Loan type missing | `Item can't be created, permanentLoanTypeId is not set, please set it in Settings - Orders - Loan type` | `title.actions.missingLoanTypeId.error` |
| Piece format mismatch | `Piece format is not compatible with purchase order line format` | `errors.pieceFormatIsNotValid` |
| Claim on non-late piece | `The piece must have "Late" status to process.` | `errors.cannotFindPiecesWithLatestStatusToProcess` |
| Claim without integration (batch) | `Unable to generate claims because no claim integrations exist for the organization.` | `errors.unableToGenerateClaimsForOrgNoIntegrationDetails` |
| Sequence out of range | `Please enter a number between {min} and {max}.` | `validation.numberRange`; UIREC-465 |
| Date range | `Start date must be earlier than or equal to the end date` / `End date must be later than or equal to the start date.` | `validation.dateRange.*` |
| Claim dates | `Selected date must be later than the current date` | `validation.dateAfter`; C423993 |
| Number generator w/o permission | `You do not have the necessary permissions to use number generator sequences. Please contact your system administrator.` | `numberGenerator.noPermission` (en.json only; SI-166) |

### Modal and dialog titles [from cases unless key given]
`Select order lines`, `Delete Holdings` (acq: `holdings.deleteModal.heading`), `Send claim`, `Select locations`, `Delete piece` (`piece.delete.heading`), `Select user`, `Add piece`, `Select Organization`, `Delay piece` (docs), `Are you sure?`, `Select instance`, `Select expected date range` (`piece.modal.expectedDateRange.heading`), `Generate numbers` (`numberGenerator.generateNumbers`), `Receiving note`, `Order closed`, `Unreceiving piece`.

### Frequent button labels
`Actions`, `Receive`, `Quick receive`, `Add piece`, `Save & close`, `Cancel`, `Edit`, `Confirm`, `Continue`, `Expect`, `Unreceive`, `Bind pieces`, `Bind`, `Receive & load more`, `Keep Holdings`, `Delete Holdings`, `Export results (CSV)`, `Create another`.

---

## Capability Sets (Eureka) and permissions

From docs `_index.md` → Capabilities and ui-receiving `package.json`:

| Capability set | Legacy display name | Notes |
|---|---|---|
| `Data - UI-Receiving - View` | Receiving: View | receive/check-in/expect endpoints, routing lists read, piece history; enough to receive pieces with connected items (UIREC-479) |
| `Data - UI-Receiving - Edit` | Receiving: View, edit | + bind pieces, batch status, send claims, title edit, routing list edit |
| `Data - UI-Receiving - Create` | Receiving: View, edit, create | + create titles |
| `Data - UI-Receiving - Delete` | Receiving: View, edit, delete | + delete titles |
| `Procedural - UI-Receiving - Execute` | Receiving: Export search results | CSV export |
| `Procedural - UI-Receiving Acq-Units Assignment - Execute` | Receiving: Assign acquisition units to new receiving title | |
| `Data - UI-Receiving Acq-Units Assignment - Manage` | Receiving: Manage acquisition units | |

- Claiming app needs Claiming view + Receiving view/edit; without Receiving view no pieces are listed, without Receiving edit no claim actions (UIREC-430; Cypress `claiming/user-without-receiving-*.cy.js`).
- Number generators need the NumberGenerators role/capabilities; otherwise the modal shows `numberGenerator.noPermission` (SI-166, C700856).
- Frequently co-required: `Data - UI-Orders Orders - View`, `Data - UI-Inventory Instance - View` [from cases].
- Legacy `Receiving: View` is minimal since 8.0.0 (UIREC-479); permissions were renamed in UIREC-490 — verify set names in the env.

---

## Settings that change Receiving behavior

- `Settings > Orders > Loan type` (default loan type; missing → `title.actions.missingLoanTypeId.error`); default instance status/type also required to create items (docs).
- `Settings > Orders > General > Number generator options` (Barcode / Accession number / Call number: Off / On, field not editable / On, field editable); sequences in `Settings > Service interaction > Number generator sequences` (C700856; icon hidden when Off — UIREC-460).
- `Settings > Orders > Routing` list configuration for printing routing lists (docs; acq: `routing.list.actions.printRoutingList.error`).
- Vendor organization `Claiming interval` and claiming integrations (Organizations app) drive Send claim behavior (docs, UIREC-412).
- ECS: `Settings > Orders > Network ordering > Central ordering` and `Active affiliation default` (C511231).
- POL fields: Receiving workflow, Create inventory, Claiming active/interval, Bindery active, Must acknowledge receiving note, Receipt status.

---

## Common Verification Patterns (from cases)

1. Title details verification — open title; verify Expected/Received/Unreceivable counts; act on piece; re-verify counts, status and the toast.
2. Order linkage — from title details open the POL number link; verify the PO line; verify receipt status after receive/unreceive.
3. Claiming state — set Claiming active + interval on POL; set piece state (Expected / Late / Claim delayed / Claim sent); run or wait for claiming job; verify status, `Status log` row and Item status `On order`.
4. Inventory side effect — after receive/unreceive/delete/edit holding, open the Instance and verify holdings count, item count per holding and item status.

---

## Claiming / Late Status Timing Rules (C436495, C436793, C436794, C436736 and more)

> Requires POL `Claiming active` + `Claiming interval` (days). `POST /orders-storage/claiming/process` (timer every 6 h, or run manually) evaluates and flips statuses.

- **Late via Claiming interval**: Expected piece whose `Expected receipt date` + interval has passed becomes `Late` after the job runs. POL receipt status unaffected; Item stays `On order`.
- **Late via Claim sent expiry**: `Claim sent` piece becomes `Late` once the claim expiry date passes and the job runs (ongoing POL shows `Ongoing`).
- **Delay claim vs interval**: a passed `Delay claim` date makes the piece `Late` even if the interval has not separately expired (C436794; Karate "Late After Delay Claim Date Passed") — docs disagree, see Known Gaps.
- No `Expected receipt date` → piece stays `Expected` (Karate `piece-status-transitions-claiming.feature` scenario 1).
- `Claiming active` unchecked → status never changes (C436808).
- Received or Unreceivable before the date → stays Received / Unreceivable (C436739, C436743, C436801).
- Re-applying `Delay claim` / `Send claim` without status change must update `statusUpdatedDate` and use the new interval (MODORDERS-1494, fixed mod-orders 13.1.7 Trillium CSP#1; C1602555). Same-days re-delay from Edit piece still open (UIREC-518).
- `Send claim` with integration creates one Export manager job per organization integration (UIREC-412; Cypress `claiming/send-claim-action-*.cy.js`); without integration the piece is just set to `Claim sent`. Internal/external notes entered in Send claim are saved on the piece, also for a new piece (UIREC-450, C1602438).
- Claiming interval must be > 0 when claiming is active (ErrorCodes `claimingConfigInvalid`).
- In all Late scenarios the Item status stays `On order`.

## Binding Pieces (Bind pieces / Bound items) — C476838, C476880 and family

> Requires POL `Bindery active` = Yes, only allowed for Physical or P/E Mix with Create inventory = Instance, Holding, Item and Independent workflow (ErrorCodes `orderFormatIncorrectForBindaryActive`, `createInventoryIncorrectForBindaryActive`, `receivingWorkflowIncorrectForBindaryActive`; Karate `bind-piece.feature`). Not shown when bindery inactive (C476877).

- Only `Received`, not-yet-bound pieces are listed (BE: `piecesMustHaveReceivedStatus`); sorted by expected receipt date (UIREC-392).
- Bind page: `Barcode`, `Call number`, `Material type*`, `Permanent loan type*`, `Permanent location*`; table of eligible pieces; `Bind` disabled until required fields + ≥ 1 piece.
- Success: toast `Item <b>{barcode}</b> created successfully`; new item `In process` in `Bound items`; items of bound pieces become `Unavailable`; pieces get `Bound` flag and leave the default Received view (UIREC-353).
- Requests on bound pieces' items: Transfer / Do not transfer / Cancel; Transfer moves all requests chronologically; if one user has several, only the oldest is moved and the others cancelled (UIREC-354; C476882, C477535, C477550).
- Bound piece shows `Item details (original)` = first item ever created; "original" stays the first item after repeated bind/unbind (UIREC-371, C506655). Manually checking/unchecking `Bound` does not change item status (C506703).
- Removing a piece from the bound item: `Bound pieces data` `X` → `Are you sure?` / `Remove this piece from the bound item?` (C494018, C502960).
- Non-unique barcode → no holding is created (C773251). Existing holding selected → no new holding (UIREC-391). Two items can be bound in the same holding (C543761).
- Holdings are not deleted or merged by binding.

## Abandoned Holdings (C503243, C503244, C934313-C934320, C1273162, C1273175)

- Delete Holdings decision is centralized in `POST /orders/holding-detail` (MODORDERS-1371) and reused by Receiving (UIREC-350/491), unopen order and change instance connection.
- Changing a piece's holding/location in Edit piece **or** in the full-screen Receive form, when the old holding would have no other POLs, pieces or items, shows the `Delete Holdings` modal (`Cancel` / `Keep Holdings` / `Delete Holdings`) (UIREC-350, fixed 8.0.0).
- Deleting a piece that leaves its holding empty prompts the holdings/item delete choice (en.json `piece.delete.deleteHoldingsAndItem.message`); C503243 reported auto-deletion without prompt — see Known Gaps.
- Holdings that still contain an item or received piece, or that are linked to the POL with Independent workflow, are not offered for deletion (C934314, C934315, UIREC-484 comment, C1273162).
- Full-screen batch: holding created on one row is not offered to other rows, including on later batches (UIREC-487, C813585).
- Edited locations appear on the POL after receiving from full screen (C478259, C543782, C543784).

## Batch Receiving on the Full-Screen "Receive" View (C813585)

- More than 50 expected pieces → `Select expected date range` modal (`Start date`/`End date` required, `Required!`); empty range → `The list contains no items`.
- Loads 50 at a time by `Expected receipt date`; subheader `{count} of {totalRecords} receiving pieces · {startDate} - {endDate}`; `Receive & load more` until the last batch (`Receive`).
- Per-row Create new holding / existing holding; Delete Holdings modal may appear mid-batch.
- Final commit → `Receiving successful`.
- ECS member tenant: `Receive & load more` follows `receivingTenantId` filtering (UIREC-486); `Create item` disabled from member tenant (UIREC-394).

## Number Generators for Barcode / Accession Number / Call Number (C700856)

- Modes per field `Off` / `On, field not editable` / `On, field editable`; `Generate numbers` icon on Item details and per row in the Receive table; `Off` hides icon and the Actions column (UIREC-460).
- Help text `The selected sequences will be used to generate new numbers. Select "Generate numbers" to go ahead.` (`numberGenerator.generateHelpText`); sequences `Barcode sequence`, `Accession number sequence`, `Call number sequence`, `Accession and call number sequence`.
- `Use the same generated number for accession number and call number` forces one shared value.
- Verify values in both the Received table and the Edit piece view after receiving.

## Piece Sequence (UIREC-465, UIREC-466; Karate `piece-sequence-numbers.feature`; C844209)

- Edit piece shows `Sequence` `[n] of {count}` (`piece.sequence`, `piece.sequence.scope`); new piece defaults to highest + 1 and deleted pieces still count (10 created, 2 deleted → next is 11 of 11).
- Lowering a sequence shifts the pieces in between up by one; raising it shifts them down by one; values stay unique within the title.
- Out-of-range value blocks save with `Please enter a number between {min} and {max}.`; success toast shows old → new.
- Inventory item order is not affected (docs → Reordering pieces).

## Piece ↔ Item Field Synchronization (C959222, C959224, C432301-C432304)

- `Display summary`, `Copy number`, `Enumeration`, `Chronology`, `Barcode`, `Call number`, `Accession number` exist on piece and item. After receipt, "Piece details" is a frozen snapshot; "Item details" mirrors the live item. Edits on the item side update the piece's Item details only.
- Edits to a received piece write only changed fields to the item (MODORDERS-1331).
- Unreceive keeps these fields on the piece (UIREC-483).
- `Display to public` requires `Display on holding` (BE `pieceDisplayOnHoldingsIsNotConsistent`; C464324).

## Routing Lists (docs → Routing pieces; acq: `routing.list.*`)

- Only for Physical / P/E Mix POLs; one routing list per physical piece; title shows `This title has routing.` (`title.hasRouting`) and an expanded `Routing lists` accordion.
- Name must be unique (`Name must be unique`); Print all routing lists needs routing configuration in Settings.
- Regression: saving a routing list on a duplicated POL (UIREC-512); app no longer closes after closing the Routing list page (UIREC-380).

---

## ECS / Consortia (C468235 family, C471514-C471517, C473256-C473259, C511231)

**Location lookup with `Affiliation` dropdown** (shared with orders.md): Receiving `Location` facet and `POL number look-up` modal; dropdown only in Central tenant, lists all tenants, Location or Holding from a member tenant both resolve (C471514-C471517, C473256-C473259).

**`Active affiliation default`** (C511231): per-member setting giving a member/central segmented control in `Search & filter`; Vendor `Organization look-up` returns only that tenant's organizations and has no Affiliation dropdown. Do not conflate with the Location lookup.

**Piece-level ECS rules:**
- Piece form and full-screen Receive have an `Affiliation` dropdown (UIREC-365); holding names shown for member holdings (UIREC-386).
- Pieces are shown only when the user is affiliated with the piece's tenant: central shows all affiliated tenants, member shows only the active one (UIREC-390, MODORDERS-1174); POL receipt status counts only pieces the user can see (MODORDERS-1179).
- Unreceivable pieces: `Select holdings` and `Affiliation` not editable (UIREC-361). `Create item` disabled from member tenant in full-screen Receive (UIREC-394). Connected item link disabled when item is in another tenant (UIREC-387).
- Bound items from other members are shown (UIREC-421); central may pick location affiliation on the Bind screen when central ordering is on (UIREC-363).
- Received pieces are counted on reopen of a synchronized order (MODORDERS-1421; Karate `consortia/features/received-pieces-counted-when-reopening-synchronized-order.feature`).
- Export with item data in a member tenant fixed (UIREC-509).

---

## Key Business Rules for Test Cases

1. Receiving is available only for Open or Closed orders; Pending orders are not receivable. (docs `_index.md` → Order status)
2. Receiving against a Closed or Cancelled order shows the `Order closed` confirmation before receive, also for Quick receive. (UIREC-497; `piece.confirmReceiving.*`)
3. Piece lifecycle: Expected → Received (Quick receive / Receive), reversible via Unreceive; Expected → Unreceivable, reversible via Expect. (PieceFormActionButtons/constants.js; C423617, C436833)
4. `Must acknowledge receiving note` blocks receive until `Continue` is clicked in the `Receiving note` dialog. (docs; `title.isAcknowledged`)
5. Receiving changes a connected item from `On order`/`Order closed` to `In process`; items in any other status keep it. (`piece.itemStatus.info`; C367971)
6. Unreceive sets the connected item to `On order` and keeps it; the piece keeps Barcode/Call number/Accession number. (`piece.actions.unreceive.message`; UIREC-483)
7. Package POLs cannot be received directly; a title must be added first. (docs → Creating a receiving title)
8. POL receipt status recalculates automatically (Partially / Fully Received; Ongoing for ongoing orders). (Karate `pieces-batch-update-status.feature`; C436793)
9. Auto-close can trigger from Receiving when receipt and payment are complete. (MODORDERS-1170; Karate `create-order-payment-not-required-fully-receive.feature`)
10. Quick receive receives a single piece from Edit piece; full-screen Receive handles many pieces and per-row edits. (docs → Receiving a piece)
11. Check in is a separate step after receiving. (docs)
12. Claiming status changes only happen with POL `Claiming active` checked and a claiming interval > 0. (C436808; ErrorCodes `claimingConfigInvalid`)
13. A passed `Delay claim` date makes the piece `Late` without waiting for the claiming interval. (C436794; Karate `piece-status-transitions-claiming.feature`) — docs disagree, see Known Gaps
14. Only Received, not-yet-bound pieces can be bound. (C476838, C476880; ErrorCodes `piecesMustHaveReceivedStatus`)
15. Delete Holdings choice appears when a piece delete or holding change would leave a holding with no other POLs, pieces or items; holdings with other content are never offered. (UIREC-350; C503244, C934314, C934316)
16. Full-screen Receive pages in batches of 50 after a Start/End expected-date range when the title has > 50 expected pieces; a holding created on one row is not offered to other rows. (UIREC-461, UIREC-487; C813585)
17. After receipt, Piece details is a frozen snapshot and Item details mirrors the live item. (C959222)
18. Two ECS mechanics exist: Location-lookup `Affiliation` dropdown vs. `Active affiliation default` toggle. (C471516, C471517, C511231)
19. Adding a piece with Create item / new holding fails with `The piece was <b>not saved</b>` and creates no Holdings/Item when encumbrance limits are enforced and exceeded or the budget is not Active. (MODORDERS-1415; C1250459, C1250460)
20. With Synchronized workflow the last piece cannot be deleted (`errors.lastPiece`); Independent workflow allows it. (UIREC-481; C594357, C1273175)
21. With Synchronized workflow and Pending order, `Add piece` is hidden and the API rejects the piece. (UIREC-362, MODORDERS-1220; C627458)
22. With Synchronized workflow, adding or deleting a piece changes POL quantity and location (warning `title.changePieceQuantityMessage`); Independent does not. (docs → Adding an expected piece)
23. The Save & close dropdown depends on piece status: Late hides `Mark late`; Received offers only Save and create another / Unreceive / Delete; Unreceivable offers Save and create another / Expect / Delete; new piece has no `Send claim`. (PieceFormActionButtons/constants.js; C844841, C656338)
24. `Send claim` asks for a future `Claim expiry date`; with an org claiming integration it creates an Export manager claim job per integration, without one it just sets `Claim sent`. (UIREC-412; C692110, C423993)
25. Internal/external notes from Send claim are saved to the piece, also for a new piece. (UIREC-450; C1602438)
26. Re-applying Delay claim / Send claim without status change updates `statusUpdatedDate` and the new interval. (MODORDERS-1494; C1602555)
27. A piece without `Expected receipt date` never becomes Late via the claiming job. (Karate `piece-status-transitions-claiming.feature`)
28. Received or Unreceivable pieces keep their status when claim/delay dates pass. (C436739, C436743, C436801)
29. Claiming batch job runs every 6 hours (`POST /orders-storage/claiming/process`); tests may trigger it via API. (mod-orders-storage ModuleDescriptor)
30. Bindery active requires Physical or P/E Mix, Create inventory = Instance, Holding, Item and Independent workflow. (ErrorCodes; Karate `bind-piece.feature`)
31. Binding creates an `In process` item; items of bound pieces become `Unavailable`; requests can be transferred (oldest per requester kept, others cancelled). (UIREC-353, UIREC-354; C476882, C477535)
32. Bound piece shows `Item details (original)` = first item ever created, stable over repeated bind/unbind; toggling `Bound` manually does not change item status. (UIREC-371; C506655, C506703)
33. Sequence: new piece = highest + 1 (deleted pieces count); changing it reshuffles the pieces in between; out-of-range blocks save. (UIREC-465, UIREC-466; C844209)
34. Editing a received piece writes only changed fields to the item and never unsuppresses it. (MODORDERS-1331, MODORDERS-1316; C784497)
35. Deleting an expected piece deletes its connected item; deleting a received piece keeps the item. (docs → Deleting a piece)
36. `Remove from package` deletes the title's pieces, items and empty related holdings; it is not offered for non-package or foreign-tenant titles and fails when pieces are Received. (UIREC-448, UIREC-452, UIREC-454; ErrorCodes `existingReceivedPiecesTitleRemoval`; C736743, C770416)
37. Unopening an order deletes unreceived pieces in every non-received status (Expected, Late, Claim sent, Claim delayed, Unreceivable). (MODORDERS-1295; C736710)
38. Piece `Expected receipt date` inherits from the POL. (MODORDERS-1322; C805752)
39. ECS: users see only pieces of tenants they are affiliated with (central: all affiliations, member: active only). (UIREC-390, MODORDERS-1174)
40. A user with only a UI-Receiving capability set (view/edit/create/delete) can receive pieces with connected items. (UIREC-479)
41. Acq units: a user outside the title's acq unit cannot receive, unreceive or change status of its pieces; title acq-unit assign/edit needs the dedicated capability sets. (C436813, C436815-C436818, C430248, C430254)
42. Received accordion hides bound pieces by default (`Bound`/`Not bound` filter); this default is intentional. (UIREC-264, UIREC-449)

---

## Existing Automated Coverage

**Karate** (`folio-integration-tests/acquisitions/src/main/resources/thunderjet/`, runners `OrdersApiTest`, `OrdersCriticalPathApiTest`, `OrdersExtendedApiTest`, `ConsortiaOrdersApiTest`):
- `mod-orders/features/`: `piece-status-transitions.feature`, `piece-status-transitions-claiming.feature`, `pieces-batch-update-status.feature`, `piece-deletion-restriction.feature`, `piece-sequence-numbers.feature`, `piece-item-synchronization.feature`, `piece-audit-history.feature`, `piece-batch-job.feature`, `bind-piece.feature`, `holding-detail.feature`, `unlink-title.feature`, `poline-claiming-interval-checks.feature`, `receive-piece-against-(non-)package-pol.feature`, `receive-20-pieces.feature`, `receive-pieces-delete-empty-holding.feature`, `receive-piece-new-holding-edit.feature`, `change-location-when-receiving-piece.feature`, `unreceive-piece-and-check-order-line.feature`, `add-piece-to-cancelled-order.feature`, `no-side-effect-with-failed-piece-operation.feature`, `independent-acquisitions-unit-for-ordering-and-receiving.feature`, `retrieve-titles-with-honor-of-acq-units.feature`
- Reusables: `mod-orders/reusable/receive-piece*.feature`, `unreceive-piece*.feature`, `create-piece*.feature`, `claim-pieces.feature`, `update-pieces-batch-status.feature`, `verify-piece.feature`
- Consortia: `consortia/features/pieces-api-test-for-cross-tenant-envs.feature`, `received-pieces-counted-when-reopening-synchronized-order.feature`; claims export: `mod-data-export-spring/features/claims-export*.feature`

**Cypress** (stripes-testing): `cypress/e2e/receiving/` (113 specs: status changes, claiming timing, bind/unbind, abandoned holdings, sequence, routing lists, acq units, export `receiving/export/`), `cypress/e2e/orders/receiving-and-check-in/`, `cypress/e2e/claiming/`, `cypress/e2e/consortia/receiving/`. Fragments: `cypress/support/fragments/receiving/` (`receiving.js`, `receivingDetails.js`, `pieceForm.js`, `bindPiecesForm.js`, `receivingStates.js`, …).

---

## Authoring style (measured 2026-07-23)

Receiving is almost purely `Functional` (~99.6%, 243/244), median ~8 steps, `User Journey` ~0%. Cases walk a real piece flow (`Title > Expected/Received accordion > Actions > Add piece / Receive / Edit piece / Delete`) and verify Inventory and often Finance side effects. Preconditions carry the acquisition setup (Open PO with ongoing/one-time POL, Synchronized/Independent workflow, Create inventory, Fund/budget state) and any Number-generator / claiming / bindery settings. Set `Type = Functional`.

---

## Known Gaps and Verification Notes

- [ ] TestRail not queried (bridge disconnected) in the 2026-11-24 refresh — no new case IDs beyond the 2026-06-17 build, the 2026-11-07 delta (C1574482-C1574484 bulk edit/delete pieces, C1453676 CSV export) and Cypress spec titles. Re-run Phase 1 when TestRail is reachable.
- [ ] **Delay claim expiry target status** — docs (`_index.md` → Claiming) say a passed delay date reverts the piece to `Late` **or `Expected`** depending on expected receipt date and interval; C436794 and Karate say `Late`. Verify in env before asserting the `Expected` branch.
- [ ] **Delete Holdings modal wording** — C503244 recorded `…no other pieces OR items connected to the related Holdings record(s)…`; current shared text (acq: `holdings.deleteModal.message`) adds `purchase order lines,`; ui-receiving also has a singular variant (`piece.actions.edit.deleteHoldings.message`). Which one the Edit piece form renders on master was not confirmed in code.
- [ ] **Piece delete leaving empty holding** — C503243 says the holding is auto-deleted without a prompt; en.json has `piece.delete.deleteHoldingsAndItem.message` (prompt). Confirm current behavior in env.
- [ ] Bind success toast in Jira/cases is `Item (barcode) created successfully`; source is `Item <b>{barcode}</b> created successfully` — assert the source form.
- [ ] Earlier "version drift" note (Unreceivable / Delay claim removed from the dropdown) was wrong for master — see the per-status dropdown section (source: constants.js, C844841).
- [ ] Earlier claim that Unreceive deletes items created on receive was wrong — Unreceive sets the item to `On order` (en.json `piece.actions.unreceive.message`).
- [ ] UIREC-518 (same-days re-delay from Edit piece doesn't update `statusUpdatedDate`) still open at refresh time.
- [ ] Eureka capability set names for claiming and number generators were not confirmed from a capability source; UIREC-490 renamed permissions — verify names in the env.
- [ ] Sequence numbers for predicted piece sets (C844201/C844209/C844210) — Serials integration angle not yet read.
- [ ] ~94 cases in section 36735 (ECS piece/title lifecycle) are still unaudited.
- Jira fixVersions for 2026 UIREC issues are mostly empty; release assignment is inferred from ui-receiving CHANGELOG (8.0.x = Trillium, 9.0.0 = Umbrellaleaf).
