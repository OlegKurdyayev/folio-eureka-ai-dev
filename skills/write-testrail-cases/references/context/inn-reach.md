---
area: inn-reach
generated_on: 2026-11-24
generator_version: manual (build-app-context method, refresh)
jira_status: ok
---

# FOLIO INN-Reach Integration — Business Logic Context

> Built: 2026-11-07 · Refreshed: 2026-11-24 · Release: Umbrellaleaf R2 2026 (master: ui-inn-reach 6.2.0 IN_PROGRESS, mod-inn-reach 4.1.0-SNAPSHOT, interface `inn-reach 1.3`; Trillium R1 2026 = ui 6.1.x / mod 4.0.x, Sunflower = ui 6.0.x / mod 3.4.x) · Builder: build-app-context method, manual
> Sources: TestRail suite 21 section 16515 "INN-Reach Integration" and 13 subsections (≈150 cases; full content read for Transactions 17499, Transactions search 17463, C844853, C380622, C1395025); GitHub folio-org/ui-inn-reach (translations/ui-inn-reach/en.json, package.json, CHANGELOG.md, src/constants/*.js, src/components/transaction/TransactionDetails/components/ActionMenu/**, src/routes/transaction/utils.js), folio-org/mod-inn-reach (descriptors/ModuleDescriptor-template.json, NEWS.md, README.md, D2IR circulation controller); folio-org/folio-integration-tests `mod-inn-reach`, `edge-inn-reach`; folio-org/stripes-testing; Jira UIINREACH / MODINREACH Done issues updated since 2024-09 (≈200 listed, ≈40 read in detail); Confluence FOLIJET pages "Inn-reach settings", "Item hold flow (Lending flow)", "Patron hold flow (Borrowing flow)", "Inn-reach app reports and slips", "Setup INN-Reach API connection with mock-server".
> [confirmed] = en.json + TestRail agree · [source] = en.json only · [cases] = TestRail only · [code] = UI/BE source · [wiki] = FOLIJET Confluence.

## What is INN-Reach

INN-Reach is III's resource-sharing network (D2IR protocol). FOLIO acts as a **local server** connected to one or more **central servers**: it contributes bib/item records, lends its items to other agencies (owning site), and borrows items for its patrons (borrowing site). Staff work in the **INN-Reach** app (transactions + check-out to borrowing site + receive shipped items) and in **Settings > INN-Reach** (connection, contribution, circulation mappings). Most state changes are driven by D2IR messages from the central server (`/inn-reach/d2ir/circ/<operation>/{trackingId}/{centralCode}`, or `/innreach/v2/...` via edge-inn-reach), so many cases need a sandbox central server, a mock server, or transactions created via API (FOLIJET wiki guides linked in case preconditions).

## Key Terms

| Term | Meaning |
|---|---|
| Central server | INN-Reach hub; configured in Settings > INN-Reach > Central server configuration (code, address, key/secret, local server code, local agency, FOLIO libraries, borrowed item loan type) |
| Local server / agency | FOLIO's identity on the central server; agencies map to FOLIO libraries/locations |
| Contribution | Sending FOLIO instances/items to the central catalog — initial (bulk, "Manage contribution") and ongoing (Kafka inventory events → outbox → scheduler) |
| Transaction | Record of one resource-sharing request. Types: **Item** (FOLIO owns the item, a remote patron requested it), **Patron** (FOLIO patron requests a remote item), **Local** (local patron, local item via central catalog) |
| ITEM_HOLD / PATRON_HOLD / LOCAL_HOLD | Initial statuses of Item / Patron / Local transactions |
| Tracking ID | Central-server ID of the transaction; key of every D2IR message |
| Virtual (temporary) records | Instance/holding/item created by FOLIO for a PATRON_HOLD; deleted on FINAL_CHECKIN or owning-site CANCEL_REQUEST [wiki] |

## Status Values

`TRANSACTION_STATUSES` in `ui-inn-reach/src/constants/transactions.js` [code]: ITEM_HOLD, PATRON_HOLD, LOCAL_HOLD, TRANSFER, ITEM_SHIPPED, ITEM_IN_TRANSIT, ITEM_RECEIVED, RECEIVE_UNANNOUNCED, RETURN_UNCIRCULATED, LOCAL_CHECKOUT, CANCEL_REQUEST, BORROWING_SITE_CANCEL, BORROWER_RENEW, CLAIMS_RETURNED, RECALL, FINAL_CHECKIN, OWNER_RENEW. Types: ITEM, PATRON, LOCAL (list labels: Item / Patron / Local, `transaction.type.*`).

Incoming D2IR circulation operations (mod-inn-reach controller `/inn-reach/d2ir/circ`): patronhold, itemshipped, cancelrequest, claimsreturned, localhold, intransit, transferrequest, cancelitemhold, receiveunshipped, returnuncirculated, itemreceived, recall, borrowerrenew, ownerrenew; plus itemhold / finalcheckin handled through the generic `POST,PUT /inn-reach/d2ir/circ/{circulationOperationName}/{trackingId}/{centralCode}` handler.

Manage contribution job statuses (`settings.manage-contribution.contribution-status.*`): Not started, In progress, Cancelled, Paused, Complete [source]. Mapping status: Valid / Invalid [source].

## Transaction lifecycle

- **Item (owning site / lending):** ITEM_HOLD → (Check out to borrowing site) ITEM_SHIPPED → (D2IR itemreceived) ITEM_RECEIVED → (D2IR intransit) ITEM_IN_TRANSIT → (Final check-in) FINAL_CHECKIN. TRANSFER after D2IR transferrequest. RECALL after a Recall request is placed on the checked-out item (final status of the transaction even if the recall is later cancelled) [wiki Item hold flow]. Owner renewal → OWNER_RENEW, borrower renewal → BORROWER_RENEW (MODINREACH-524).
- **Patron (borrowing site):** PATRON_HOLD (virtual instance/holding/item + item-level Page request created) → ITEM_SHIPPED → (Receive item) ITEM_RECEIVED, or (Receive unshipped item from PATRON_HOLD/TRANSFER) RECEIVE_UNANNOUNCED → (Check out to patron) loan, status unchanged → (Check in app / Return item) ITEM_IN_TRANSIT → (D2IR finalcheckin) FINAL_CHECKIN, virtual records deleted [wiki Patron hold flow].
  - Cancel hold (staff) → BORROWING_SITE_CANCEL, request cancelled, item goes back to lender [wiki]. Owning-site cancel (D2IR cancelrequest) → CANCEL_REQUEST, request cancelled, notice sent, then virtual records deleted (MODINREACH-574; C1395178). Remove hold → CANCEL_REQUEST (UIINREACH-262; C844853).
- **Local:** LOCAL_HOLD → Check out to local patron (sends LOCAL_CHECKOUT) / Transfer hold to another item; closing the FOLIO request by any "Closed - …" status also sends LOCAL_CHECKOUT (MODINREACH-427). RETURN_UNCIRCULATED when the item is returned without circulating.
- RECALL: FINAL_CHECKIN must be handled when the loaned item is checked in, and IN_TRANSIT messages honored, even in RECALL (MODINREACH-575/576; C1385304, C1385310).
- Cancelling the FOLIO request cancels the INN-Reach transaction and vice versa (MODINREACH-562/563).

## UI structure

- **INN-Reach app** — "Select activity" dropdown (`searchAndFilter`) at the top of Search & filter: **INN-Reach transactions** | **Check out to borrowing site** | **Receive shipped items** (`shipped-items.title.receive-shipped-items`).
  - Transactions list columns (`transaction.field.*`): Transaction time, Transaction type, Item title (Agency code), Patron name (Agency code), Transaction status (INN-Reach). Title/Patron columns sortable again from ui 6.2.0 (UIINREACH-265, MODINREACH-573; C1385302); not sortable in Trillium 6.1.x (UIINREACH-279; C1538675).
  - Search by: Item barcode, Patron barcode, Item ID, Item title, Patron ID, Patron name, Tracking ID. Filters: Transaction type, Transaction status, Central server, Patron agency, Item agency, INN-Reach patron type, INN-Reach item type (C350565–C350738).
  - Transaction detail: Transaction summary, Patron information, Item information accordions (`transaction-detail.accordion.*`); "Record last updated" source for system updates shows "System", not "Unknown user" (UIINREACH-281, ui 6.1.1).
  - Pane Actions: five CSV reports (`reports.*.label`) and "INN-Reach paging slips for <service point>" (disabled when the current service point has no ITEM_HOLD transactions [wiki]; C380622).
  - Check out to borrowing site: barcode entry → loan row with "..." menu: Loan detail, Change due date, Inn-Reach transaction detail (`check-out-borrowing-site.action.*`).
- **Settings > INN-Reach** — General: Central server configuration, INN-Reach locations; Record contribution: Contribution criteria, FOLIO to INN-Reach locations, Circulation status mapping (contribution options), Material to item type mapping, Bib transformation options, Manage contribution (Current contribution / Contribution history); Circulation mappings: Agency to FOLIO location, Central item type, Central patron type, FOLIO circulation user for INN-Reach patron type, Patron agency, INN-Reach recall user, Visible patron ID, INN-Reach paging slip templates, Pickup locations.
- All settings forms use the standard unsaved-changes modal: "Are you sure?" / "There are unsaved changes" / "Keep editing" / "Close without saving" [confirmed].

## Transaction actions — enabling conditions

From `ActionMenu/components/{Item,Patron,Local}Actions.js` [code]. Every action is visible but **disabled** outside these conditions.

| Type | Action (en.json key `transaction-detail.*.action.*`) | Enabled when |
|---|---|---|
| Item | Check out to borrowing site | status ITEM_HOLD or TRANSFER |
| Item | Transfer hold to another item | ITEM_HOLD/TRANSFER **and** hold has folioItemId and folioRequestId |
| Item | Recall item | ITEM_SHIPPED, ITEM_RECEIVED, RECEIVE_UNANNOUNCED |
| Item | Cancel hold | ITEM_HOLD or TRANSFER |
| Item | Final check-in | ITEM_RECEIVED, RECEIVE_UNANNOUNCED, ITEM_IN_TRANSIT |
| Patron | Check out to patron | ITEM_RECEIVED or RECEIVE_UNANNOUNCED **and** no folioLoanId |
| Patron | Receive item | ITEM_SHIPPED |
| Patron | Receive unshipped item | PATRON_HOLD or TRANSFER |
| Patron | Return item | ITEM_RECEIVED or RECEIVE_UNANNOUNCED |
| Patron | Cancel hold | PATRON_HOLD, ITEM_RECEIVED, RECEIVE_UNANNOUNCED **and** folioRequestId present |
| Patron | Remove hold | PATRON_HOLD **and** no folioItemId, folioRequestId, folioLoanId |
| Local | Check out to local patron | LOCAL_HOLD or TRANSFER |
| Local | Transfer hold to another item | LOCAL_HOLD or TRANSFER; shown only with `ui-requests.moveRequest` permission |
| Local | Cancel hold | LOCAL_HOLD only |

## Reports (Actions menu, CSV)

Query parameters from `src/routes/transaction/utils.js` [code], columns from `src/constants/reports.js` + `reports.*` keys:

| Report | Type / statuses | Date filter | Columns |
|---|---|---|---|
| Owning site overdue report | ITEM; ITEM_RECEIVED, BORROWER_RENEW, OWNER_RENEW, ITEM_IN_TRANSIT, ITEM_SHIPPED | loan due date < today − Minimum days overdue | INN-Reach Patron ID, INN-Reach Patron name, Item's Effective Location, Item call number, Item Barcode, Item Title, Patron Agency, Loan Due Date |
| Requested too long report | PATRON; PATRON_HOLD, TRANSFER | created and updated before today − Minimum days requested | Item Location, Item call number, Item Barcode, Item Title, Patron Agency, INN-Reach Patron ID, Date Requested |
| Returned too long report | PATRON; ITEM_IN_TRANSIT, RETURN_UNCIRCULATED | updated before today − Minimum days returned | Item Location, Item call number, Item Barcode, Item Title, Patron Home Library, Requesting Patron ID, Date Returned |
| Owning site paged too long report | ITEM + LOCAL; ITEM_HOLD, LOCAL_HOLD, TRANSFER | created **after** today − Minimum days paged (see Known Gaps) | Item HRID, Item's Effective Location, Item call number, Item Barcode, Item Title, Requesting Patron Agency, Paged Date |
| In-transit too long report | PATRON; ITEM_SHIPPED | updated before today − Minimum days shipped | Item Location, Item call number, Item Barcode, Item Title, Patron Home Library, Requesting Patron ID, Date Shipped |

No matching records → toasts "Export in progress" then "No items found." (MODINREACH-467).

## API surface (mod-inn-reach, interface `inn-reach 1.3`)

From `descriptors/ModuleDescriptor-template.json`; `permissionsRequired` in brackets.

- Transactions: `GET /inn-reach/transactions` [inn-reach.inn-reach-transactions.collection.get] (supports sortBy title / patronName since MODINREACH-573); `GET|PUT /inn-reach/transactions/{transactionId}` [inn-reach.inn-reach-transaction.item.get / .put].
- Staff actions (`POST`): `/transactions/{id}/receive-item/{servicePointId}` [inn-reach.d2ir.inn-reach-transaction.item-receive.post]; `/transactions/{id}/receive-unshipped-item/{servicePointId}/{itemBarcode}` [….receive-unshipped-item.post]; `/transactions/{itemBarcode}/check-out-item/{servicePointId}` (check out to borrowing site) [….check-out-item.post]; `/transactions/{id}/patronhold/check-out-item/{servicePointId}` [….patron.check-out-item.post]; `/transactions/{id}/localhold/check-out-item/{servicePointId}` [….localhold.check-out-item.post]; `/transactions/{id}/patronhold/cancel` [….patron-hold.cancel.post]; `/transactions/{id}/patronhold/remove` [….patron-hold.remove.post]; `/transactions/{id}/itemhold/cancel` [….item-hold.cancel.post]; `/transactions/{id}/localhold/cancel` [….localhold.cancel.post]; `/transactions/{id}/itemhold/recall` [….item-hold.recall.post]; `/transactions/{id}/patronhold/return-item/{servicePointId}` [….patron-hold.return-item.post]; `/transactions/{id}/itemhold/finalcheckin/{servicePointId}` [….item-hold.final-checkin.post]; `/transactions/{id}/itemhold/transfer-item/{itemId}` [inn-reach.d2ir.inn-reach-transaction.item.post]; `/transactions/{id}/localhold/transfer-item/{itemId}` [inn-reach.transactions.transfer-item.post].
- D2IR inbound: `POST,PUT /inn-reach/d2ir/circ/{op}/{trackingId}/{centralCode}` [inn-reach.d2ir.circulation.all]; `POST /inn-reach/d2ir/circ/verifypatron` [inn-reach.d2ir.patron-info.item.get]; `POST /inn-reach/d2ir/getbibrecord/{bibId}/{centralCode}` [inn-reach.d2ir.bib-info.item.get]; `POST /inn-reach/authentication`.
- Settings (all under `/inn-reach/central-servers/{centralServerId}/…`, GET/POST/PUT, some DELETE): contribution-criteria, item-contribution-options, material-type-mappings, libraries/location-mappings, libraries/{libraryId}/locations/location-mappings, agency-mappings, marc-transformation-options, patron-type-mappings, item-type-mappings, central-patron-type-mappings, user-custom-field-mappings, inn-reach-recall-user, visible-patron-field-configuration, paging-slip-template; plus `/inn-reach/central-servers` CRUD and `/inn-reach/locations` CRUD.
- Contribution: `POST /central-servers/{id}/contributions` (initiate), `GET|DELETE …/contributions/current`, `GET …/contributions/history`; `GET …/marc-record-transformation/{inventoryInstanceId}`; timer `POST /inn-reach/central-servers/contributions/cleanup` (outbox cleanup, MODINREACH-588).
- Slips: `GET /inn-reach/paging-slips/{servicePointId}` [inn-reach.paging-slips.collection.get].

## Permissions / Capability Sets (Eureka)

| Okapi permission (package.json `stripes.permissionSets`) | Eureka capability set |
|---|---|
| `ui-inn-reach.app.all` — INN-Reach App: All permissions | `Data - UI-Inn-Reach App - manage` (confirmed: C844853, C1543961 preconditions, which spell it `data - UI-Inn-Reach App - manage`; C1395025 differs, see Known Gaps) |
| `ui-inn-reach.settings.view` — Settings (inn-reach): All INN-Reach settings: Read-only | `Settings - UI-Inn-Reach Settings - view` (confirmed by UIINREACH-259 description: "Settings - UI-Inn-Reach Settings" has `view` and `edit`; exact casing in a Eureka UI not seen) |
| `ui-inn-reach.settings.edit` — …: Read, edit | `Settings - UI-Inn-Reach Settings - edit` (confirmed by UIINREACH-259, as above) |
| `ui-inn-reach.settings.all` (replaces `ui-inn-reach.settings.crud`) — …: CRUD | `Settings - UI-Inn-Reach Settings - manage` (confirmed: UIINREACH-259 names "Settings - UI-Inn-Reach Settings - manage" as the replacement for the Crud set; MODINREACH-542) |

Most transaction cases also need Check out, Inventory view and Users loans view capabilities (preconditions of C353659/C353939); Local "Transfer hold to another item" additionally needs `ui-requests.moveRequest` [code].

## Settings rules (FOLIJET wiki "Inn-reach settings" unless noted)

- INN-Reach locations: code exactly 5 characters, description optional [wiki]; mod-inn-reach `innReachLocationDTO.json` only enforces `maxLength: 5` (exact length is wiki-only).
- Contribution criteria [wiki]: excluded locations → items there are not contributed; an instance whose holdings are all in excluded locations is de-contributed. "Contribute but suppress" stat code → contributed, not discoverable. "Exclude" stat code → not contributed; instance de-contributed when all its items carry it. "System-owned resource" stat code → instance contributed as INN-Reach resource.
- Material to item type mapping: adding a new material type in Settings > Inventory makes the mapping status Invalid until mapped; ongoing contribution stops meanwhile (MODINREACH-539 comments).
- FOLIO to INN-Reach locations: for a library mapping, only locations of that library are listed (UIINREACH-246).
- Circulation status mapping [wiki]: unavailable statuses, non-lendable loan types / locations / material types → item contributed as Unavailable.
- FOLIO circulation user for INN-Reach patron type: user barcodes with special symbols cannot be saved; all central patron types are listed (no 10-row limit, UIINREACH-274, Trillium+).
- Pickup locations: checked → service point from the transaction payload is used; unchecked → the mapped user's default pickup service point is used; without one the transaction is created as CANCEL_REQUEST (C357039, C357046, C357048).
- Central server configuration: "Generate keypair" in edit mode regenerates only the secret; the local server key is preserved (UIINREACH-272, ui 6.1.0); the local server key must be a UUID (MODINREACH-490).
- Checkout settings are read from `/circulation/settings` (other_settings), not mod-configuration (MODINREACH-561, mod 4.0.0).

## Karate and Cypress coverage

- Karate `folio-org/folio-integration-tests/mod-inn-reach` (Vega; runner `src/test/java/org/folio/InnReachApiTest.java`, 23 `.feature` files in `src/main/resources/vega/mod-inn-reach/features/`, verified 2026-11-25): central-server(-configuration), inn-reach-location, contribution(-criteria), item-contribution-options-configuration, material-type-mapping, item-type-mapping, library-mapping, location-mapping, agency-mappings, central-patron-type-mapping, patron-type-mapping, user-custom-field-mapping, inn-reach-recall-user, marc-transformation-options-settings, marc-record-transformation, bib-info, patron-info, authentication, inn-reach-proxy, inn-reach-transaction (item/patron hold lifecycle incl. negative cases), inn-reach-circulation (all D2IR operations incl. ownerrenew, finalcheckin). Helpers in `global/transaction-helper.feature`. Uses Okapi permissions and mod-configuration.
- Karate `edge-inn-reach` (runner `EdgeInnReachApiTest.java`): central-server, inn-reach-proxy.
- Not covered by Karate: paging-slip-template, paging-slips, visible-patron-field-configuration, patronhold/remove, localhold/transfer-item, transaction sorting by title/patronName, report queries.
- Cypress `stripes-testing`: only `cypress/e2e/inn-reach-integration/verify-that-record-meta-data-is-present-on-staff-slips.cy.js` (C411863, Circulation staff slips); no INN-Reach app UI specs.

## ECS / Multi-Tenant Notes

INN-Reach is not part of ECS setups; INN-Reach cases are marked "ECS unsupported" (C844853, C380622, C1395025). UIINREACH-237 comment: INN-Reach modules are not part of the ECS setup. Eureka support started with Sunflower CSP (UIINREACH-259).

## Key Business Rules for Test Cases

1. Every transaction action is enabled only for specific type+status combinations (table above); each action has a negative case asserting it is inactive in other statuses (C350948, C350953, C350956, C350961, C353620, C353591; ActionMenu `*Actions.js`).
2. Receiving a shipped item requires the item to have a barcode; an already received item cannot be received again (C353221, C350968, C353525).
3. Receive unshipped item asks for a barcode; if it already exists on another FOLIO item the agency code is appended to the barcode and the modal "The barcode for this item already exists in FOLIO" plus toast "The barcode associated with the transaction has been updated" appear (C350910; `shipped-items.modal.barcode-augmented.heading`).
4. Final check-in (Item type) is allowed in ITEM_RECEIVED, RECEIVE_UNANNOUNCED and ITEM_IN_TRANSIT; repeating it is blocked (C353568–C353615; ItemActions.js).
5. Check out to borrowing site on an ITEM_HOLD item creates a FOLIO loan, sets the transaction to ITEM_SHIPPED, closes the request as Closed - Filled and sets the item to Checked out (C357006; wiki Item hold flow).
6. Change due date from the borrowing-site loan menu behaves like Users > Change due date, incl. "New due date is in the past" warning and loan history action "Due date changed" (C353953–C353970).
7. Contribution criteria: excluded locations and three statistical-code fields (contribute but suppress / exclude / system-owned); a code chosen in one field is not selectable in another (C345363).
8. Material to item type mapping and FOLIO to INN-Reach locations require a mapping for every row before Save is enabled (C345364, C345365).
9. Bib transformation: "Strip fields and subfields" can be saved without "Modify MARC 001 field for contributed records" (C350720).
10. Central server secrets are obfuscated after save; generating a new local key pair preserves the local server key (C350755, UIINREACH-272).
11. Ongoing contribution: item create/update/delete, bulk edit, moving holdings/items between instances must be reflected in the central catalog (de-contribute then re-contribute on move) (C503082–C506653, MODINREACH-512).
12. Pickup locations: with local pickup disabled, the request uses the patron's default pickup service point; without a default SP no request is generated and the transaction is CANCEL_REQUEST (C357039, C357046, C357048; wiki Inn-reach settings).
13. Paging slip templates: per central server, token list, preview, print, images can be saved (C356767–C1538597, MODINREACH-518).
14. Owning site overdue report CSV columns: INN-Reach Patron ID, INN-Reach Patron name, Item's Effective Location, Item call number, Item Barcode, Item Title, Patron Agency, Loan Due Date (C1543961, UIINREACH-289; `src/constants/reports.js`).
15. Owning site overdue report uses the transaction due date, which follows loan renewal and manual due-date change for ITEM transactions (MODINREACH-514).
16. Remove hold is enabled only for PATRON_HOLD with no FOLIO item, request and loan; it sets CANCEL_REQUEST (soft cancel, record kept), shows "Hold has been successfully removed", and afterwards all actions are disabled; the same item can be requested again (UIINREACH-262, MODINREACH-482; C844853).
17. Patron "Cancel hold" requires a linked FOLIO request and sets BORROWING_SITE_CANCEL; the linked request is cancelled and the lender must do final check-in (PatronActions.js; wiki Patron hold flow).
18. Owning-site cancel (D2IR cancelrequest) on a PATRON_HOLD/TRANSFER transaction cancels the FOLIO request, sends the request cancellation notice, then deletes virtual instance/holding/item and unlinks request and patron (MODINREACH-574; wiki Patron hold flow).
19. Cancel hold from the INN-Reach app on ITEM_HOLD/TRANSFER sets CANCEL_REQUEST and closes the FOLIO request as Closed - Cancelled; checking the item in afterwards makes it Available, not In transit (MODINREACH-562/563).
20. Item hold request for an item that already has an open request, or is in Long missing, Declared lost, Aged to lost, Lost and paid, Awaiting pickup, Missing, Paged, Claimed returned, Withdrawn, Awaiting delivery, In process (non-requestable), Unavailable or Unknown status creates the transaction as CANCEL_REQUEST with no FOLIO request (wiki Item hold flow).
21. When the FOLIO request for an ITEM_HOLD is rejected by circulation rules, FOLIO sends owning-site cancel and re-contributes the item so the central catalog shows it as available again (MODINREACH-485).
22. Items in Paged, In transit, Declared lost, Withdrawn, Awaiting pickup, Aged to lost or Long missing status are contributed with itemCircStatus "Not Available" regardless of settings (MODINREACH-123, MODINREACH-469, MODINREACH-526).
23. An instance with no remaining contributable items (location excluded or stat code excluded) is de-contributed from the central catalog (MODINREACH-539, MODINREACH-499; wiki Inn-reach settings).
24. Closing the FOLIO request of a LOCAL_HOLD with any "Closed - …" status sends LOCAL_CHECKOUT to the central server (MODINREACH-427).
25. Renewal from the owning side (D2IR ownerrenew) sets OWNER_RENEW; renewal by the borrower sets BORROWER_RENEW; the ownerrenew due date applies only if it is earlier than the loan-policy renewal date (MODINREACH-524).
26. Placing a Recall request on a checked-out ITEM transaction item (ITEM_SHIPPED/ITEM_RECEIVED) sets RECALL; the status stays RECALL even if the recall is cancelled; final check-in still works from RECALL (wiki Item hold flow; MODINREACH-575/576).
27. Check-in of a returned Item-hold item via the Check in app must move the transaction to FINAL_CHECKIN (MODINREACH-557, fixed with MODINREACH-572).
28. PATRON_HOLD final check-in removes patron ID/name, request, loan and item links from the transaction and deletes the virtual instance; the same item can then be requested again (wiki Patron hold flow).
29. Patron "Check out to patron" creates the loan per circulation rules, closes the request as Closed - Filled and keeps the transaction status (ITEM_RECEIVED / RECEIVE_UNANNOUNCED) (wiki Patron hold flow).
30. Receive item / Receive unshipped item with the pickup service point selected puts the item in Awaiting pickup (modal "Awaiting pickup for a request"); with another service point the item goes In transit (modal "In transit") (wiki Patron hold flow; `shipped-items.modal.*`).
31. Hold and transit slips printed from Receive item must contain requester data for a user with only the INN-Reach app capability set (MODINREACH-595/543); hold slip shows patron last, first, middle name and barcode, not preferred name, department or patron group (MODINREACH-538).
32. Paging slip token `item.effectiveCallNumber` includes prefix, call number, suffix, volume, enumeration, chronology, copy number (UIINREACH-255, MODINREACH-503).
33. Paging slips for a service point print only ITEM_HOLD transactions whose items belong to that service point; the action is disabled when there are none (wiki reports and slips; C380622).
34. Transactions list sorts by Item title and Patron name without error (UIINREACH-265, MODINREACH-573; C1385302).
35. System-made transaction updates show "System" as the last-updated source (UIINREACH-281).
36. A cancelled INN-Reach patron-hold request shows its title as "-" with no hyperlink in the Requests app (UIREQ-1365; C1395025).

## Edge cases from bugs

- Initial item HRID above 2,147,483,647 in Settings > Inventory > HRID handling blocks virtual item creation for PATRON_HOLD (MODINREACH-530) — check environment before PATRON_HOLD tests.
- Request cancellation reason with id `941c7055-04f8-4db3-82cb-f63965c1506f` must exist for owning-site cancel to cancel the linked request (wiki Patron hold flow).
- Owning site paged too long "Minimum days paged" filter was ignored (UIINREACH-236, ui 6.0.0).
- Long-URI 414 errors on paging slips with many requests fixed (MODINREACH-531).
- Settings save with only the settings-manage capability failed on Sunflower BF (MODINREACH-542, environment issue).
- Mock-server environments cannot test contribution; do not click "Initiate contribution" there (wiki Setup INN-Reach API connection with mock-server).

## Known Gaps / Items to Verify

- [ ] Capability set names for the Settings sets are confirmed only by the UIINREACH-259 description (not in Cypress dictionary, which has no INN-Reach entries, nor in a case); exact casing on a Eureka env not seen. C1395025 writes the app set as "Data - UI - Inn-Reach - Manage" while C844853/C1543961 use "data - UI-Inn-Reach App - manage" — verify on a Eureka env.
- [ ] Owning site paged too long report: code sends `createdDate > today − (minDays − 1)` (`getParamsForOwningSitePagedTooLongReport`), i.e. transactions created within the last N days; the wiki "Inn-reach app reports and slips" asks the same question; UIINREACH-236 expected older-than semantics. Confirm with PO before asserting.
- [ ] Wiki Item hold flow says Final check-in becomes active at ITEM_RECEIVED; code also enables it in RECEIVE_UNANNOUNCED and ITEM_IN_TRANSIT. Code wins.
- [ ] Wiki Patron hold flow: Cancel hold is clickable after check-out to patron but does nothing; renewal of an Item-hold loan from Users does not change the transaction status ("to be investigated").
- [ ] Owning site overdue report and manually set due dates: wiki (2025-12) says such transactions were missing; MODINREACH-514 (later) fixed sync — re-verify.
- [ ] MODINREACH-627/625/624 (InventoryViewClient HRID performance, Spring Boot 4.1) still in review/awaiting deployment at refresh date.
- [ ] Generic success toasts in settings cases say only "Successful toast message appears" — use the exact texts from the table below.
- [ ] Many old cases have Test Group 4/5/7 (Obsolete/Draft/Edge) — do not copy their metadata.
- [ ] docs.folio.org has no INN-Reach page (sitemap and search, 2026-11-24); FOLIJET Confluence is the only end-user-level description.

---

## Exact UI Texts

> From `folio-org/ui-inn-reach` `translations/ui-inn-reach/en.json` at HEAD (2026-11-07; re-checked 2026-11-24: all keys unchanged; en_US.json identical except it lacks `reports.patronName`). `{x}` are placeholders; `<b>`/`<strong>` render bold. Use verbatim.

### Toast Messages

| Key | Text | Source |
|---|---|---|
| `validate.required` | Required | [source] |
| `modal.heading.areYouSure` | Are you sure? | [source] |
| `modal.message.unsavedChanges` | There are unsaved changes | [source] |
| `modal.confirmLabel.keepEditing` | Keep editing | [source] |
| `modal.cancelLabel.closeWithoutSaving` | Close without saving | [source] |
| `settings.central-server-configuration.callout.connectionProblem.get` | Unable to get data due to a module error. Please try again. If a problem persists please contact your system administrator | [source] |
| `settings.central-server-configuration.callout.deleted` | Central server configuration was successfully deleted | [source] |
| `settings.central-server-configuration.callout.failedToDelete.delete` | Failed to delete a central server configuration | [source] |
| `settings.central-server-configuration.callout.connectionProblem.delete` | Unable to delete due to a module error. Please try again. If a problem persists please contact your system administrator | [source] |
| `settings.central-server-configuration.callout.connectionProblem.post` | Unable to create data due to a module error. Please try again. If a problem persists please contact your system administrator | [source] |
| `settings.central-server-configuration.callout.connectionProblem.put` | Unable to updated data due to a module error. Please try again. If a problem persists please contact your system administrator | [source] |
| `settings.central-server-configuration.callout.invalidData` | Invalid central server data | [source] |
| `settings.central-server-configuration.callout.notUnique` | Some of the values entered must be unique | [source] |
| `settings.central-server-configuration.create.success` | Central Server configuration created successfully | [source] |
| `settings.central-server-configuration.update.success` | Central Server configuration updated successfully | [source] |
| `settings.central-server-configuration.edit.modal-heading.updateLocalServerKeyConfirmation` | Update Local Server Key Confirmation | [source] |
| `settings.central-server-configuration.edit.modal-message.updateLocalServerKeypair` | You have updated the Local Server Key/Secret pair. INN-Reach circulation functions will be interrupted until this information is updated by the Central Server administrator. Click “Confirm” to save your changes and download a JSON file with the new key and secret values. | [source] |
| `settings.central-server-configuration.edit.modal-button.confirm` | Confirm | [source] |
| `settings.confirmDeleteModal.central-server-configuration.heading` | Confirm deletion of central server configuration | [source] |
| `settings.confirmDeleteModal.central-server-configuration.message` | Central server configuration {name} will be removed. | [confirmed] |
| `settings.confirmDeleteModal.central-server-configuration.confirm` | Delete | [source] |
| `settings.confirmDeleteModal.central-server-configuration.cancel` | Cancel | [source] |
| `settings.contribution-criteria.create.success` | Contribution Criteria settings created successfully | [source] |
| `settings.contribution-criteria.update.success` | Contribution Criteria settings updated successfully | [source] |
| `settings.folio-to-inn-reach-locations.create-edit.create.success` | FOLIO to INN-Reach locations settings created successfully | [source] |
| `settings.folio-to-inn-reach-locations.create-edit.update.success` | FOLIO to INN-Reach locations settings updated successfully | [source] |
| `settings.contribution-options.create.success` | Circulation status mapping settings created successfully | [source] |
| `settings.contribution-options.update.success` | Circulation status mapping settings updated successfully | [source] |
| `settings.material-type-mapping.update.success` | Material type mappings updated successfully | [source] |
| `settings.agency-to-folio-locations.callout.connection-problem.put` | There was a problem and your update was not saved: {message}, status code {code} | [source] |
| `settings.agency-to-folio-locations.create.success` | The INN-Reach Agency to FOLIO Location Mapping for {name} has been created successfully | [source] |
| `settings.agency-to-folio-locations.update.success` | The INN-Reach Agency to FOLIO Location Mapping for {name} has been updated successfully | [source] |
| `settings.bib-transformation.create.success` | Bib transformation options created successfully | [source] |
| `settings.bib-transformation.update.success` | Bib transformation options updated successfully | [source] |
| `settings.bib-transformation.callout.connection-problem.post` | There was a problem and your creation was not saved: {message}, status code {code} | [source] |
| `settings.manage-contribution.initaiate.failed` | Initiate contribution failed | [source] |
| `settings.manage-contribution.initaiate.success` | Contribution successfully initiated | [source] |
| `settings.manage-contribution.cancel.failed` | Cancel contribution failed | [source] |
| `settings.manage-contribution.cancel.success` | Contribution canceled successfully | [source] |
| `settings.manage-contribution.item-mapping-type-status.invalid` | Invalid | [source] |
| `settings.manage-contribution.label.error` | Error | [source] |
| `settings.central-item-type.create.success` | Central item type settings created successfully | [source] |
| `settings.central-item-type.update.success` | Central item type settings updated successfully | [source] |
| `settings.central-patron-type.create.success` | Central patron type settings created successfully | [source] |
| `settings.central-patron-type.update.success` | Central patron type settings updated successfully | [source] |
| `settings.patron-agency.create.success` | Patron agency settings created successfully | [source] |
| `settings.patron-agency.update.success` | Patron agency settings updated successfully | [source] |
| `settings.inn-reach-recall-user.create.success` | INN-Reach recall user settings created successfully | [source] |
| `settings.inn-reach-recall-user.update.success` | INN-Reach recall user settings updated successfully | [source] |
| `settings.visible-patron-id.create.success` | Visible patron id settings created successfully | [source] |
| `settings.visible-patron-id.update.success` | Visible patron id settings updated successfully | [source] |
| `settings.paging-slip-template.preview-modal-header` | Preview of INN-Reach Paging Slip | [source] |
| `settings.paging-slip-template.create.success` | INN-Reach paging slip template settings created successfully | [source] |
| `settings.paging-slip-template.update.success` | INN-Reach paging slip template settings updated successfully | [source] |
| `settings.pickup-locations.update.success` | Local pickup locations updated successfully | [source] |
| `shipped-items.modal.no-transaction.heading` | No matching INN-Reach transaction found for this item | [source] |
| `shipped-items.modal.barcode-augmented.heading` | The barcode for this item already exists in FOLIO | [confirmed] |
| `shipped-items.modal.hold.heading` | Awaiting pickup for a request | [confirmed] |
| `shipped-items.modal.hold.comment` | <strong>Patron comments:</strong> {comment} | [source] |
| `shipped-items.modal.hold.message` | Place <strong>{title} ({materialType})</strong> (Barcode: {barcode}) on Hold Shelf at <strong>{pickupServicePoint}</strong> for request. | [source] |
| `shipped-items.modal.transit.heading` | In transit | [confirmed] |
| `shipped-items.modal.transit.message` | Route <strong>{title} ({materialType})</strong> (Barcode: {barcode}) to <strong>{servicePoint}</strong>. | [source] |
| `shipped-items.modal.message.no-transaction` | This item may not have been checked out by the owning site. Please locate the transaction for this item using the search "INN-Reach transactions" function on the landing page and perform a "Receive unshipped item" action. | [source] |
| `shipped-items.modal.message.barcode-augmented` | The agency code for this item has been appended to the barcode. You will need to use this updated barcode to perform circulation actions on this transaction | [confirmed] |
| `shipped-items.modal.button.close` | Close | [source] |
| `shipped-items.callout.connection-problem.put.receive-shipped-item` | There was a problem receiving the item | [confirmed] |
| `check-out-borrowing-site.modal.message` | No INN-Reach transaction could be found for this item. | [source] |
| `check-out-borrowing-site.callout.connection-problem.post.check-out-borrowing-site` | There was a problem checking out the item | [source] |
| `check-out-borrowing-site.callout.success.post.check-out-borrowing-site` | Item has been successfully checked out | [confirmed] |
| `unshipped-item.modal.title` | Receive unshipped item | [confirmed] |
| `unshipped-item.callout.connection-problem.post.receive-unshipped-item` | There was an error updating the barcode for the transaction | [confirmed] |
| `unshipped-item.callout.success.post.receive-unshipped-item` | The barcode associated with the transaction has been updated | [confirmed] |
| `receive-item.callout.success.post.receive-item` | Item has been successfully received | [confirmed] |
| `recall-item.callout.connection-problem.post.recall-item` | There was a problem recalling the item | [source] |
| `recall-item.callout.success.post.recall-item` | Item has been successfully recalled | [source] |
| `reports.callout.export-in-progress` | Export in progress | [source] |
| `reports.callout.no-items-found` | No items found. | [source] |
| `reports.modal.title.owning-site-overdue-report` | Owning site overdue report | [source] |
| `reports.modal.title.requested-too-long-report` | Requested too long report | [confirmed] |
| `reports.modal.title.returned-too-long-report` | Returned too long report | [source] |
| `reports.modal.title.paged-too-long-report` | Owning site paged too long report | [source] |
| `reports.modal.title.in-transit-too-long-report` | In-transit too long report | [source] |
| `reports.modal.field.minimum-days-overdue` | Minimum days overdue | [source] |
| `reports.modal.field.minimum-days-requested` | Minimum days requested | [confirmed] |
| `reports.modal.field.minimum-days-returned` | Minimum days returned | [source] |
| `reports.modal.field.minimum-days-paged` | Minimum days paged | [source] |
| `reports.modal.field.minimum-days-shipped` | Minimum days shipped | [source] |
| `reports.modal.button.save-and-close` | Save & close | [source] |
| `loan.callout.connection-problem.get.loan` | There was a problem loading the loan | [source] |
| `cancel-hold.callout.connection-problem.get.cancellation-reasons` | Could not retrieve cancellation reasons! | [source] |
| `cancel-hold.callout.connection-problem.post.cancel-hold` | There was a problem canceling the item | [confirmed] |
| `cancel-hold.callout.success.post.cancel-hold` | Item has been successfully canceled | [confirmed] |
| `return-patron-hold-item.callout.success.post.return-patron-hold-item` | Item has been successfully returned | [source] |
| `return-patron-hold-item.callout.connection-problem.post.return-patron-hold-item` | There was a problem returning the item | [source] |
| `final-check-in-item-hold.callout.success.post.final-check-in-item-hold` | Item has been successfully checked in | [confirmed] |
| `final-check-in-item-hold.callout.connection-problem.post.final-check-in-item-hold` | There was a problem checking in the item | [source] |
| `transfer-hold.modal.label` | Select item | [source] |
| `transfer-hold.modal.title` | Other items in {title} | [source] |
| `transfer-hold.modal.result-count` | {count, number} {count, plural, one {Record found} other {Records found}} | [source] |
| `transfer-hold.modal.column.barcode` | Barcode | [source] |
| `transfer-hold.modal.column.item-status` | Item status | [confirmed] |
| `transfer-hold.modal.column.request-queue` | Request queue | [source] |
| `transfer-hold.modal.column.location` | Location | [source] |
| `transfer-hold.modal.column.material-type` | Material type | [confirmed] |
| `transfer-hold.modal.column.loan-type` | Loan type | [confirmed] |
| `transfer-hold.modal.list-of-items.notFound` | No items found | [source] |
| `transfer-hold.callout.success.post.transfer-hold` | Item has been successfully transferred | [source] |
| `transfer-hold.callout.connection-problem.post.transfer-hold` | There was a problem transferring the item | [source] |
| `remove-hold.callout.success.post.remove-hold` | Hold has been successfully removed | [confirmed] |
| `remove-hold.callout.connection-problem.post.remove-hold` | There was a problem removing the hold | [source] |
| `paging-slips.callout.connection-problem.get.pagingSlip` | There was a problem loading the paging slips | [source] |
| `paging-slips-template.callout.connection-problem.get.pagingSlipTemplate` | There was a problem loading the paging slip templates | [source] |

### Action / Button Labels

| Key | Text | Source |
|---|---|---|
| `searchAndFilter` | Select activity | [source] |
| `shipped-items.title.receive-shipped-items` | Receive shipped items | [source] |
| `check-out-borrowing-site.title.check-out-borrowing-site` | Check out to borrowing site | [source] |
| `check-out-borrowing-site.placeholder.scan-or-enter-item-barcode` | Scan or enter item barcode | [source] |
| `check-out-borrowing-site.button.end-session` | End session | [source] |
| `check-out-borrowing-site.action.transaction-detail` | Inn-Reach transaction detail | [source] |
| `transaction-detail.patron-type.action.check-out` | Check out to patron | [source] |
| `transaction-detail.patron-type.action.receive-item` | Receive item | [source] |
| `transaction-detail.patron-type.action.receive-unshipped-item` | Receive unshipped item | [confirmed] |
| `transaction-detail.patron-type.action.return-item` | Return item | [source] |
| `transaction-detail.patron-type.action.cancel-hold` | Cancel hold | [source] |
| `transaction-detail.patron-type.action.remove-hold` | Remove hold | [confirmed, C844853] |
| `transaction-detail.item-type.action.check-out` | Check out to borrowing site | [confirmed] |
| `transaction-detail.item-type.action.transfer-hold` | Transfer hold to another item | [source] |
| `transaction-detail.item-type.action.recall-item` | Recall item | [source] |
| `transaction-detail.item-type.action.final-check-in` | Final check-in | [source] |
| `transaction-detail.local-type.action.check-out` | Check out to local patron | [source] |
| `transaction-detail.local-type.action.transfer-hold` | Transfer hold to another item | [source] |
| `unshipped-item.placeholder.enter-item-barcode` | Enter item barcode | [source] |
| `unshipped-item.button.submit` | Submit | [source] |
| `reports.owning-site-overdue.label` | Owning site overdue report (CSV) | [source] |
| `reports.requested-too-long.label` | Requested too long report (CSV) | [source] |
| `reports.returned-too-long.label` | Returned too long report (CSV) | [source] |
| `reports.paged-too-long.label` | Owning site paged too long report (CSV) | [source] |
| `reports.in-transit-too-long.label` | In-transit too long report (CSV) | [source] |
| `settings.manage-contribution.button.initiate-contribution` | Initiate contribution | [source] |
| `settings.manage-contribution.button.cancel-contribution` | Cancel contribution | [source] |
| `settings.manage-contribution.button.pause-contribution` | Pause | [source] |
| `settings.manage-contribution.tooltip.check-mappings` | Valid mappings are required to initiate contribution | [source] |
| `settings.manage-contribution.history.list.no.history` | No contribution history records | [source] |
| `check-out-borrowing-site.callout.connection-problem.get.transactions` | Unable to get data due to a module error. Please try again. If a problem persists please contact your system administrator | [source] |
