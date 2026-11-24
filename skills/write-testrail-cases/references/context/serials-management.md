---
area: serials-management
generated_on: 2026-11-24
generator_version: manual (build-app-context method, full refresh)
jira_status: ok
---

# FOLIO Serials Management — Business Logic Context

> Built: 2026-11-07 · Refreshed: 2026-11-24 · Release: Umbrellaleaf (R2 2026) master — ui-serials-management 2.2.0 / mod-serials-management 2.2.0 "IN PROGRESS" at HEAD; last released line = Trillium (R1 2026) ui 2.1.1 / mod 2.1.1 (CHANGELOG.md, NEWS.md) · Builder: build-app-context method, manual
> Sources: TestRail suite 21 section 34766 "Serials Management" (9 cases) + 34767 "Publication patterns" (25) + 111083 "Prediction pattern templates" (1) + 34768 "Predicted piece sets" (7) = 42 cases; preconditions of all cases and full steps of C451603 read.
> GitHub: folio-org/ui-serials-management (translations/ui-serials-management/en.json, package.json, src/), folio-org/mod-serials-management (service/src/main/okapi/ModuleDescriptor-template.json, grails-app controllers/domain, NEWS.md), folio-org/stripes-testing, folio-org/folio-integration-tests.
> Jira: UISER / MODSER Done/Closed issues resolved 2024-09 → 2026-07 (186 issues listed, ~60 read in detail). Note: fixVersion is empty on most UISER/MODSER issues — release attribution comes from CHANGELOG.md / NEWS.md.
> Docs: docs.folio.org/docs/acquisitions/serials/ (capability table, workflow).
> Tags: [source] = en.json key at HEAD; [case Cxxx] = TestRail case text; [code path] = repo file.

## What is Serials Management

The **Serials** app links a **serial** record to a PO line (Orders) and defines a **publication pattern** (ruleset: publication cycle, omission or combination rules, chronology and enumeration labels, display-summary template). From the active pattern the user **generates predicted pieces** (a predicted piece set), then **generates receiving pieces** that are created in Receiving (mod-orders `POST /orders/pieces-batch`) for the PO line's title. **Publication pattern templates** (backend: model rulesets) store reusable patterns that pre-fill the pattern form. Settings hold pick lists (refdata) used by textual enumeration and other dropdowns.

## Key Terms

| Term | Definition (source) |
|---|---|
| Serial | Record linking PO line, notes, status (Active / Closed), publication patterns and piece sets (mod `Serial.groovy`; docs.folio.org) |
| Publication pattern (ruleset) | `SerialRuleset`: status Active / Draft / Deprecated, recurrence, omission, combination, templateConfig; Pattern ID = `rulesetNumber` (mod `SerialRuleset.groovy`) |
| Publication pattern template (model ruleset) | `ModelRuleset`: name (required), description, example label, status Active / Closed, embedded serialRuleset (mod `ModelRuleset.groovy`) |
| Predicted piece set | Saved set of internal pieces generated from a ruleset; startDate (required), numberOfCycles, note, initial/continuation metadata (mod `PredictedPieceSet.groovy`) |
| Internal piece | Recurrence piece, omission piece (omitted) or combination piece (several issues combined) (mod `domain/org/olf/internalPiece/`) |
| Receiving piece | Piece in mod-orders/Receiving created from a predicted piece; its id is stored back on the internal piece (`receivingPieces`) (ui `GenerateReceivingModal.js`) |
| Display summary | Label rendered from the pattern Template + tokens; becomes the receiving piece `displaySummary` (ui `GenerateReceivingModal.js`) |

## Record Hierarchy

```
PO line (mod-orders) ──1:1── Serial (Active | Closed)
                               ├── Notes
                               ├── Publication patterns (SerialRuleset) — max 1 Active, n Draft, n Deprecated
                               │      └── Predicted piece sets (each bound to one ruleset)
                               │             └── Internal pieces (recurrence / omitted / combined)
                               │                    └── receivingPieces[] → Receiving piece ids (mod-orders)
Publication pattern templates (ModelRuleset) — independent of serials; used to pre-fill the pattern form
```

## Workflow

Serial (link POL, Status Active) → Add publication pattern (Active / Draft / Deprecated; optionally pre-filled from a template) → Generate predicted pieces (Start date, Number of years/cycles, starting enumeration values or "Follow on from the last piece in a previous set", Preview) → Predicted piece set ("Date generated", Start date, Note, pieces with Issue count / publication date / Display summary / Receiving pieces count) → Generate receiving pieces (Time between publication and receipt, Supplement, optional Create item / Display in holding / Display to public, Affiliation + Holding/Location) → pieces visible in **Receiving** for the POL title (C451603).

### Status values

| Field | Values | Source |
|---|---|---|
| Serial status (`serialStatus`) | Active, Closed | mod `Serial.groovy` @Defaults |
| Publication pattern status (`rulesetStatus`) | Active, Draft, Deprecated | mod `SerialRuleset.groovy` @Defaults |
| Template status (`modelRulesetStatus`) | Active, Closed | mod `ModelRuleset.groovy` @Defaults |
| Cycle time unit | Day, Week, Month, Year | mod `Recurrence.groovy` |
| Recurrence pattern type | Day, Week, Month Date, Month Weekday, Year Date, Year Weekday, Year Month Weekday | mod `RecurrenceRule.groovy` |
| Omission time unit / rule types | Day, Week, Month, Issue / Day, Day Month, Day Week, Day Weekday, Day Week Month, Week, Week Month, Month, Issue, Issue Week, Issue Week Month, Issue Month | mod `OmissionRule.groovy` |
| Combination time unit / rule types | Issue only / Issue, Issue Week, Issue Week Month, Issue Month | mod `CombinationRule.groovy` |
| Enumeration numeric format / sequence | Ordinal, Number, Roman / Reset, Continuous | mod `EnumerationNumericLevelTMRF.groovy` |
| Chronology formats | Weekday: Slice Lower, Slice Upper, Full Lower, Full Upper; Month day: Ordinal, Number; Month: Full, Slice, Number; Year: Full, Slice | mod `ChronologyDateTMRF.groovy`, `ChronologyMonthTMRF.groovy`, `ChronologyYearTMRF.groovy` |
| Receiving piece format | Physical only (select disabled) | ui `GenerateReceivingModalForm.js` |

Pattern status transitions (mod `SerialRulesetController.groovy`): saving / activating a pattern as Active automatically sets the serial's previous Active pattern to Deprecated; `replaceAndDeprecate` saves the new pattern and deprecates the source; `replaceAndDelete` saves the new pattern and deletes the source.

## Permissions and Eureka capability sets

| Okapi permission (ui package.json displayName) | permissionName | Eureka capability set (docs.folio.org; TestRail preconditions) |
|---|---|---|
| Serials: Search & view serials | `ui-serials-management.serials.view` | Data - UI-Serials-Management Serials - View (C622592) |
| Serials: Edit serials | `ui-serials-management.serials.edit` | Data - UI-Serials-Management Serials - Edit (C622593) |
| Serials: Manage serials | `ui-serials-management.serials.manage` | Data - UI-Serials-Management Serials - Manage (C709265) |
| Serials: Edit publication patterns | `ui-serials-management.rulesets.edit` | Data - UI-Serials-Management Rulesets - Edit (C622594) |
| Serials: View publication pattern templates | `ui-serials-management.modelrulesets.view` | Data - UI-Serials-Management Modelrulesets - View (C1322911) |
| Serials: Manage publication pattern templates | `ui-serials-management.modelrulesets.manage` | Data - UI-Serials-Management Modelrulesets - Manage (docs) |
| Serials: Search & view predicted pieces | `ui-serials-management.predictedpieces.view` | Data - UI-Serials-Management Predictedpieces - View (docs) |
| Serials: Create predicted pieces | `ui-serials-management.predictedpieces.edit` | Data - UI-Serials-Management Predictedpieces - Edit (C446151) |
| Serials: Delete predicted pieces | `ui-serials-management.predictedpieces.delete` | Data - UI-Serials-Management Predictedpieces - Delete (C451603) |
| Settings (Serials): View pick lists and values | `ui-serials-management.picklists.view` | Data - UI-Serials-Management Picklists - View (docs) |
| Settings (Serials): Manage pick lists and values | `ui-serials-management.picklists.manage` | Data - UI-Serials-Management Picklists - Manage (docs) |
| (number generator for Pattern ID) | — | Data - UI-Service-Interaction NumberGenerator - View or Manage (docs; C446129) |
| (receiving piece creation, mod-orders) | `orders.pieces.collection.post` | see Known Gaps — C451603 uses "Data - Orders Pieces Collection - create", C553044 / UISER-241 use "Procedural - Orders Pieces Collection - Execute" |

Permission composition (ui package.json): serials.edit ⊃ serials.view; serials.manage ⊃ serials.edit; rulesets.edit ⊃ serials.edit + `serials-management.modelRulesets.view` (template picker); predictedpieces.view ⊃ serials.view; predictedpieces.edit / .delete ⊃ predictedpieces.view; modelrulesets.manage ⊃ modelrulesets.view; picklists.manage ⊃ picklists.view.

## API surface (mod-serials-management, interface `serials-management`)

| Method + path | permissionsRequired |
|---|---|
| GET /serials-management/serials, /serials/{id} | serials-management.serials.collection.get / .item.get |
| POST, PUT /serials-management/serials(/{id}) | serials-management.serials.item.post / .item.put |
| DELETE /serials-management/serials/{id} | serials-management.serials.item.delete |
| GET /serials-management/rulesets, /rulesets/{id} | serials-management.rulesets.collection.get / .item.get |
| POST /serials-management/rulesets | serials-management.rulesets.item.post |
| DELETE /serials-management/rulesets/{id} | serials-management.rulesets.item.delete |
| POST /rulesets/{id}/active, /deprecated, /draft | serials-management.rulesets.set-active / set-deprecated / set-draft.execute |
| POST /rulesets/{id}/replaceAndDeprecate, /replaceAndDelete | serials-management.rulesets.replace-deprecate / replace-delete.execute |
| GET /serials-management/predictedPieces, /predictedPieces/{id} | serials-management.predictedPieceSets.collection.get / .item.get |
| PUT /serials-management/predictedPieces/{id} | serials-management.predictedPieceSets.item.put (UI stores receiving piece ids here) |
| POST /serials-management/predictedPieces/generate (preview, not saved) | serials-management.predictedPieceSets.transient.generate |
| POST /serials-management/predictedPieces/create (saved set) | serials-management.predictedPieceSets.generate |
| DELETE /serials-management/predictedPieces/{id} | serials-management.predictedPieceSets.item.delete |
| GET /serials-management/modelRulesets, /modelRulesets/{id} | serials-management.modelRulesets.collection.get / .item.get |
| POST /serials-management/modelRulesets | serials-management.modelRulesets.item.post |
| POST /modelRulesets/{id}/replaceAndDelete (template edit) | serials-management.modelRulesets.replace-delete.execute |
| DELETE /serials-management/modelRulesets/{id} | serials-management.modelRulesets.item.delete |
| GET/POST/PUT/DELETE /serials-management/refdata… | serials-management.refdata.* |
| GET /serials-management/locales | serials-management.locales.collection.get |

Source: `service/src/main/okapi/ModuleDescriptor-template.json`. No PUT exists for rulesets or model rulesets — edits are always replace-and-delete / replace-and-deprecate. UI also calls mod-orders `POST /orders/pieces-batch[?createItem=true]`, `orders/titles`, `holdings-storage/holdings`, `locations` (ui `src/constants/endpoints.js`).

## Key Business Rules for Test Cases

### Serial record
1. Serial Status values are Active and Closed (mod `Serial.groovy`); search filter "Status" uses `serialStatus.value`, plus a "PO line" filter (ui `routes/SerialsRoute/SerialsRoute.js`).
2. Serials search covers id, description and `orderLine.title`; results are paginated 50 per page; Title sort = `orderLine.title,id` (ui `SerialsRoute.js`; UISER-185, UISER-242).
3. Serials and Piece sets search do not load records until a search or filter is entered (UISER-189).
4. "New" serial button is shown only with Serials: Edit serials (UISER-164; ui `SerialsRoute.js` IfPermission).
5. Delete serial: Actions → Delete shown only with Serials: Manage serials; confirm modal "Delete serial" / `serials.deleteSerial.confirmMessage`; a serial with one or more publication patterns is not deleted and shows `serials.deleteSerial.error.hasPublicationPattern` (UISER-197; C709265; ui `SerialView.js`).
6. Closed serial: "Add publication pattern", "Generate predicted pieces" and "Generate receiving pieces" are disabled; Edit and viewing piece sets stay available (UISER-157; ui `PublicationPattern.js`, `SerialView.js`, `PieceSetView.js`).
7. "Generate predicted pieces" in serial Actions also requires Serials: Create predicted pieces (disabled otherwise) (ui `SerialView.js`).
8. Serial view shows "Serial has no active publication patterns" when no Active pattern exists; the accordion badge counts Active + Draft patterns; Deprecated patterns are listed separately (`ruleset.deprecatedPublicationPatterns`) (ui `PublicationPattern.js`, `SerialView.js`).
9. PO line linking uses the find-po-line plugin; for a package POL the Title selector must search and return more than 10 titles (UISER-244); DELETE serial returned 500 on C-env once — not reproducible after retest (MODSER-140).

### Publication patterns
10. Only one Active pattern per serial: the backend rejects >1 Active (`only.one.active.ruleset`) and saving a new Active pattern auto-deprecates the previous Active one (mod `Serial.groovy`, `SerialRulesetController.groovy`; docs.folio.org).
11. Recurrence validation: number of issue rules must equal "No. of issues published per cycle" (`rules.must.equal.issues`); each rule ordinal must be within 1..period (`ordinal.not.in.range`) (mod `Recurrence.groovy`, `RecurrenceRule.groovy`).
12. A pattern has either omission rules or combination rules, never both: "Add omission rule" is disabled with tooltip `ruleset.addOmissionDisabledTooltip` when a combination exists and vice versa (`ruleset.addCombinationDisabledTooltip`) (en.json; mod `PredictedPieceSetController.groovy` applies omission OR combination).
13. Combination rules support only time unit "Issue" and types Issue / Issue Week / Issue Week Month / Issue Month (mod `CombinationRule.groovy`).
14. Edit vs Copy and deprecate: Actions → Edit is disabled once the pattern has any predicted piece set (`replaceAndDelete` returns 405 "Cannot delete a serial ruleset which has been used to generate a predicted piece set"); "Copy and deprecate" is always available and deprecates the source (UISER-153; C619001, C619002; ui `RulesetView.js`; mod `SerialRulesetController.groovy`).
15. Edit / Copy and deprecate actions are hidden (not just disabled) without Serials: Edit publication patterns (UISER-153 QA; ui `RulesetView.js`).
16. After saving a new pattern the publication pattern view opens (UISER-210; C451603 step 5).
17. Preview is possible without any label template, once status and publication cycle are filled and no rule is half-completed (UISER-70, MODSER-153 — Trillium).
18. "Days of publication, per cycle" are sorted in cycle order (start → end of year) in view and edit (UISER-240 — Trillium).
19. Enumeration levels: "Add level" before entering data must create Level 1, Level 2… correctly and typing must not lose focus (UISER-250, UISER-259 — Trillium 2.1.1).
20. Textual enumeration uses pick-list (refdata) values and they are re-populated in edit/copy screens (UISER-191, UISER-204, MODSER-99).
21. Applying a pattern template replaces the form content (`patternTemplate`); the chosen template stays visible in edit/copy views (UISER-182 QA).
22. Pattern ID comes from the number generator; without one the backend falls back to sequential numbering (MODSER-53). Legacy patterns without rulesetNumber had no link (UISER-224, script fix).

### Predicted piece sets
23. Number of years/cycles defaults to 1; total predicted issues are capped at 366, enforced by `validate.allowedCycles` (max cycles = floor(366 / issues per year)) (UISER-179; ui `PiecesPreviewModalForm.js`; en.json `numberOfCycles.infoPopover`).
24. Same start date as an existing set for the serial → warning under Start date (`pieceSets.overlappingDates.warning`) and confirm modal "Confirm generation of overlapping piece sets" with "Generate" to continue; the check is an exact start-date match (UISER-148, UISER-174; ui `PiecesPreviewModal.js`).
25. When the pattern was used before, the form offers "Follow on from the last piece in a previous set", pre-filling Start date (previous start + number of cycles) and next enumeration values as plain integers (UISER-91, UISER-179 QA). Ramsons defect MODSER-135 closed as not reproducible in Sunflower/Trillium.
26. Omitted pieces show as "Omitted" in preview and set but create no receiving piece (MODSER-46, MODSER-58; ui `GenerateReceivingModal.js` filters omission pieces).
27. Omission / combination rules by Issue apply per cycle in multi-cycle prediction (issue 11 omitted in each of 3 cycles → issues 11, 22, 33) (MODSER-151, MODSER-152).
28. A combined piece consumes all combined issue numbers (Jan/Feb combined as Issue 1 → March = Issue 3), whereas an omitted issue does not consume a number (MODSER-101). Combined piece shows "Combined pieces: {length}" (en.json `pieceSets.combinedPieces`).
29. A combined last piece must not cause an error (MODSER-103); a combined piece's date = date of its first recurrence piece (MODSER-54; ui `GenerateReceivingModal.js`).
30. Multiple continuous enumeration levels number independently; v.114 must not jump to v.115 inside one year (MODSER-62, MODSER-117).
31. Issue publication date equals the configured day in tenants behind UTC (no −1 day), also in the "follow on" Start date (UISER-211).
32. Delete predicted piece set: needs Serials: Delete predicted pieces; disabled once any piece has receiving pieces; backend returns 405 "Cannot delete a predicted piece set which contains receiving pieces" (UISER-155, UISER-178; C619003, C451603 step 16; mod `PredictedPieceSetController.groovy`).
33. Piece sets search covers id, `ruleset.rulesetNumber` and note; columns include Title and Date generated (ui `PieceSetsRoute.js`).

### Receiving pieces
34. Receiving pieces are created only when the user submits "Generate receiving pieces"; the report of pieces appearing overnight was attributed to another user (EBSCOAdmin) retrying, not to the system (UISER-241).
35. "Generate receiving pieces" is disabled when the serial is not Active or has no linked POL; the submit button stays disabled until "Time between publication and receipt (days)" (required) is filled (ui `PieceSetView.js`, `GenerateReceivingModalForm.js`; C451603 steps 13–14).
36. Receiving piece fields: receiptDate = piece publication date + interval days; displaySummary = piece label; format Physical; poLineId/titleId from the serial's POL; supplement / displayOnHolding / displayToPublic default false (ui `GenerateReceivingModal.js`; C451603: interval 9 → expected receipt on the 10th).
37. All pieces are created in one call `POST /orders/pieces-batch` (createItem as query param); then the set is updated with receiving ids and each row's "Receiving pieces" count becomes 1 (UISER-194; C451603 step 15).
38. "Create item" checkbox appears only when POL Create inventory = "Instance, Holding, Item"; "Display in holding" / "Display to public" appear only when the POL has holdings (ui `GenerateReceivingModalForm.js`; C553044).
39. POL with no locations/holdings shows `pieceSets.noOrderLineLocationsOrHoldings` and pieces are created without location (C451603 step 13); a POL with one location pre-fills holding/location and disables Affiliation (ui `GenerateReceivingModal.js`).
40. Re-generating for a set that already has receiving pieces shows `pieceSets.receivingPiecesAlreadyExistWarning` (ui `GenerateReceivingModalInfo.js`).
41. Failure shows `pieceSets.generateReceivingErrorMessage` + backend message, and no pieces are created (UISER-118, UISER-241).
42. Creating receiving pieces needs the mod-orders permission `orders.pieces.collection.post`; Receiving UI permissions do not include it (UISER-213 — workaround: assign the capability).
43. ECS with central ordering: an Affiliation select sets `receivingTenantId`; holdings/locations are filtered by that tenant; orders created in a member tenant can generate receiving pieces there (UISER-161, UISER-205).
44. Verify generated pieces in the **Receiving app** (title, expected receipt date, display summary/enumeration/chronology) — the effect's real destination (C451603 steps 17–18).

### Publication pattern templates (model rulesets)
45. "Templates" search/sort is visible only with Serials: View publication pattern templates; search covers name and description; filter "Status" (`modelRulesetStatus.value`); default sort name ascending (UISER-225, UISER-230, UISER-232; ui `RouteSwitcher.js`, `TemplatesRoute.js`).
46. New / Edit / Copy / Delete template actions need Serials: Manage publication pattern templates; Edit = replaceAndDelete (copy into form, delete old on save); Copy opens a pre-filled Create form (UISER-229, UISER-231, UISER-233, UISER-234, MODSER-141).
47. Template name is required; status required (Active / Closed); the template's embedded pattern is saved without an owner serial (mod `ModelRuleset.groovy`, `ModelRulesetController.groovy`; MODSER-105).
48. A user without the templates view permission must still open the Serials app without errors (UISER-235).

## Edge cases from bugs (regression candidates)

- Ramsons → Sunflower migrated patterns with combination preview/generate without errors and with correct labels (MODSER-109, MODSER-110).
- Preview with combination rules works (UISER-170).
- First or next piece omitted does not break generation (MODSER-58).
- Template search/filter checkbox shows as selected (UISER-249, UISER-252).
- Panes keep width between Serials / Piece sets / Templates tabs (UISER-246); correct pane header (UISER-247).
- Locations list loads more than 10 records (UISER-186).

## Existing coverage

- TestRail (42 cases): Smoke/Critical Path list maintained in UISER-261 (C446127, C622592–C622594, C446128, C446129, C446130, C446151, C451603 …). Non-Draft: C446127 Create a serial record; C709265 Delete a serial record; C622592/C622593/C622594 permission cases; C446128 Link a PO line; C446152/C446153/C446154 serial search; C446129, C446130, C1346385 add pattern; C619001 Edit unused pattern; C619002 Copy and deprecate; C1322911 Add prediction pattern template; C446151 Generate predicted pieces; C451603 Generate receiving pieces; C553044 … with items; C613211 combined pieces; C619003 Delete predicted piece set; C446155, C446156 navigation/search of piece sets. C446131–C446150 (frequencies, omissions, combinations) have no preconditions/refs (likely Draft).
- Shared step 42 (create monthly pattern → generate predicted pieces) is reused by C451603.
- Cypress (stripes-testing): `cypress/e2e/fse/ui/serials-ui.cy.js` (TC195636 app opens), `cypress/e2e/fse/api/serials-api.cy.js` (TC195523 GET serials by status), `cypress/support/fragments/serials/serials.js`, `cypress/support/api/serials.js`. `cypress/e2e/orders/receiving-and-check-in/serials-receiving.cy.js` is an Orders/Receiving spec, not the Serials app.
- Karate (folio-integration-tests): no serials-management project exists. Backend has its own integration tests (MODSER-83, MODSER-125).

## Known Gaps / Items to Verify

- [ ] Orders pieces capability for receiving piece creation: C451603 "Data - Orders Pieces Collection - create" vs C553044 / UISER-241 "Procedural - Orders Pieces Collection - Execute" — confirm on Eureka env.
- [ ] C553044 lists "Data - UI-Receiving - View, Edit, Create" and C451603 "Data - UI-Receiving - create"; whether Receiving UI capability is needed beyond verification in Receiving is unconfirmed.
- [ ] Capability set names for Predictedpieces View, Modelrulesets Manage, Picklists View/Manage come from docs.folio.org (Trillium docs), not from cases.
- [ ] UISER-157: Jira description lists expected Closed-serial behaviour; resolution comment not read — verified only against UI code (disabled buttons).
- [ ] MODSER-135 (follow-on enumeration) closed Not reproducible; MODSER-117 volume-change calculation partially explained as by-design in comments — confirm expected numbering with PO before asserting.
- [ ] Settings → Serials pick lists: no TestRail cases; behaviour from en.json only.
- [ ] Draft frequency cases C446131–C446150 steps not read.
- [ ] `pieceSets.receivingPiecesAlreadyExistWarning` exists in en.json but not yet in en_US.json (Lokalise lag).
- [ ] Jira fixVersion fields are mostly empty for UISER/MODSER; release per change taken from CHANGELOG/NEWS.

---

## Exact UI Texts

> From `folio-org/ui-serials-management` `translations/ui-serials-management/en.json` at HEAD (2026-11-24). `{x}` are placeholders; `<b>`/`<strong>` render bold. Use verbatim.

### Toast / Callout Messages

| Key | Text | Source |
|---|---|---|
| `refdata.deleteRefdataValue.confirmMessage` | <strong>{name}</strong> will be deleted. | [source] |
| `refdata.deleteRefdataValue.errorMessage` | <strong>Error: {label}</strong> was not deleted. {error} | [source] |
| `refdataCategory.deleteRefdataCategory.errorMessage` | <strong>Error</strong>: Pick list <strong>{label}</strong> was not deleted. | [source] |
| `serials.deleteSerial.confirmMessage` | Serial <strong>{serial}</strong> will be deleted | [source] |
| `serials.deleteSerial.error.hasPublicationPattern` | Serial was not deleted because it has one or more publication patterns. | [source] |
| `serials.deleteSerial.success` | Serial deleted: <strong>{serial}</strong> | [source] |
| `serials.deleteSerial.error.noDeleteSerialBackendError` | <strong>Serial was not deleted:</strong> {message} | [source] |
| `templates.deleteTemplate.confirmMessage` | Publication pattern template <strong>{name}</strong> will be deleted | [source] |
| `templates.deleteTemplate.success` | Publication pattern template deleted: <strong>{name}</strong> | [source] |
| `templates.deleteTemplate.error` | <strong>Publication pattern template was not deleted:</strong> {message} | [source] |
| `ruleset.confirmGeneration` | Confirm generation of overlapping piece sets | [source] |
| `pieceSets.countReceivingGenerated` | <strong>{count} receiving pieces were generated.</strong> Please check this total is as expected. The pieces can now be viewed in the Receiving app. | [source] |
| `pieceSets.generateReceivingErrorMessage` | <strong>Error:</strong> No receiving pieces were generated.  | [source] |
| `pieceSets.receivingPiecesAlreadyExistWarning` | Warning: Receiving pieces already exist for this predicted piece set. See {link} for more details. | [source] |
| `pieceSets.overlappingDates.warning` | Warning: A predicted piece set with the start date <strong>{startDate}</strong> already exists for the serial <strong>{serialName}</strong> | [source] |
| `pieceSets.overlappingDates.dialog` | A predicted piece set with the start date <strong>{startDate}</strong> already exists for the serial <strong>{serialName}</strong>. {br} Generating a new set will result in some overlap between piece sets. If this is acceptable select "Generate" to continue. | [source] |
| `pieceSets.deletePredictedPieceSet.confirmMessage` | Are you sure you wish to delete this predicted piece set? | [source] |
| `pieceSets.noOrderLineLocationsOrHoldings` | There are no locations or holdings for the linked POL and receiving pieces will be created with no location or holding. To add a location or holding please update the PO line. | [source] [case C451603] |
| `pieceSets.generateReceivingInfo` | Selecting 'Generate receiving pieces' will create new pieces in the Receiving app. Please wait while the pieces are generated. This window will close when the process is complete. | [source] |

### Modal / Pane Titles

| Key | Text | Source |
|---|---|---|
| `serials.newSerial` / `serials.editSerial` / `serials.deleteSerial` | New serial / Edit serial / Delete serial | [source] |
| `rulesets.newPublicationPattern` | New publication pattern | [source] [case C451603] |
| `ruleset.editPublicationPattern` / `ruleset.copyAndDeprecatePublicationPattern` | Edit publication pattern / Copy and deprecate publication pattern | [source] |
| `ruleset.generatePredictedPieces` / `ruleset.previewPredictedPieces` | Generate predicted pieces / Preview predicted pieces | [source] |
| `pieceSets.generateReceivingPieces` | Generate receiving pieces | [source] [case C451603] |
| `pieceSets.deletePredictedPieceSet` | Delete predicted piece set | [source] [case C451603] |
| `templates.newTemplate` / `templates.editTemplate` / `templates.deleteTemplate` | New publication pattern template / Edit pattern template - {name} / Delete publication pattern template | [source] |
| `refdataCategory.deleteRefdataCategory` / `refdata.deleteRefdataValue` | Delete pick list category / Delete refdata value | [source] |

### Buttons and actions

New, Edit, Delete, Save, Cancel, Close, Actions (`new`, `edit`, `delete`, `save`, `cancel`, `close`, `actions`); Add publication pattern (`serials.addPublicationPattern`); Copy and deprecate (`ruleset.copyAndDeprecate`); Copy (`templates.copy`); Preview (`ruleset.preview`); Generate (`ruleset.generate`); Cancel generation (`cancelGeneration`); Add omission rule / Add combination rule (`ruleset.addOmission`, `ruleset.addCombination`); Add chronology label / Add enumeration label / Add level / Add value (`ruleset.addChronologyLabel`, `ruleset.addEnumerationLabel`, `ruleset.addLevel`, `ruleset.addValue`); Add note (`serials.addNote`); Select PO line / Replace PO line / Remove PO line (`poLine.selectPOLine`, `poLine.replacePOLine`, `poLine.removePOLine`); View in inventory (`poLine.viewInInventory`). [source]

### Field labels (key ones)

| Key | Text |
|---|---|
| `ruleset.cycleLength` / `ruleset.timeUnit` / `ruleset.numberOfTimeUnit.month` | Cycle length / Time unit / Number of months |
| `ruleset.numberOfIssuesPerCycle` | No. of issues published per cycle |
| `ruleset.daysOfPublicationPerCycle` / `ruleset.dayFormat` | Days of publication, per cycle / Day format |
| `ruleset.day.monthRange` / `ruleset.day.weekdayRange` / `ruleset.week.monthRange` / `ruleset.week.yearRange` / `ruleset.month.yearRange` / `ruleset.issue.nRange` | Day (1-31) / Day (Mon-Sun) / Week (1-4) / Week (1-52) / Month (Jan-Dec) / Issue (1-n) |
| `ruleset.omissionRuleType` / `ruleset.combinationRuleType` | Omission rule type / Combination rule type |
| `ruleset.numberOfIssuesToCombine` / `ruleset.totalNumberOfIssuesToCombine` / `ruleset.firstIssue` | Number of issues to combine / Total number of issues to combine / First issue |
| `ruleset.chronologyFormat` / `ruleset.enumerationFormat` | Chronology format / Enumeration format |
| `ruleset.numberOfUnits` / `ruleset.format` / `ruleset.sequence` | No. of units / Format / Sequence |
| `ruleset.enumeration.numberFormat.number` / `.ordinal` / `.roman` | Number (1, 2, 3) / Ordinal (1st, 2nd, 3rd) / Roman numerals (I, II, III) |
| `ruleset.template` / `ruleset.template.tokens` / `ruleset.labelTemplate` | Template / Template tokens / Label template |
| `ruleset.startDate` / `ruleset.numberOfCycles` | Start date / Number of years/cycles |
| `ruleset.valuesToUseForFirstIssue` / `ruleset.followOnFromPrevious` | Values to use for the first issue / Follow on from the last piece in a previous set |
| `ruleset.issueCount` / `ruleset.issuePublicationDate` / `pieceSets.displaySummary` | Issue count / Issue publication date / Display summary |
| `pieceSets.dateGenerated` / `pieceSets.pieceSetGenerated` / `pieceSets.receivingPieces` / `pieceSets.omitted` | Date generated / Piece set generated / Receiving pieces / Omitted |
| `pieceSets.timeBetweenPublicationAndReceipt` | Time between publication and receipt (days) |
| `pieceSets.pieceFormat` / `pieceSets.supplement` / `pieceSets.createItem` | Piece format / Supplement / Create item |
| `pieceSets.displayInHolding` / `pieceSets.displayToPublic` / `pieceSets.affiliation` / `pieceSets.holding` / `pieceSets.location` | Display in holding / Display to public / Affiliation / Holding / Location |
| `templates.modelRulesetName` / `templates.modelRulesetStatus` / `templates.exampleLabel` | Pattern template name / Pattern template status / Example label |
| `patternTemplate` | Pattern template (replaces content in the form below with pre-defined values) |

### Error / validation / empty-state messages

| Key | Text | Source |
|---|---|---|
| `validate.withinRange` | Must be within range {minValue} and {maxValue} | [source] |
| `validate.notNegative` | Cannot be negative | [source] |
| `validate.wholeNumber` | Must be a whole number | [source] |
| `validate.mustEqual` | Must equal {value} | [source] |
| `validate.greaterThanOne` | Must be greater than 1 | [source] |
| `validate.allowedCycles` | For this publication pattern it is only possible to generate issues for up to {maxValue} years/cycles at one time to avoid generating very large piece sets | [source] |
| `ruleset.addOmissionDisabledTooltip` | Omission rules cannot be added because a combination rule exists | [source] |
| `ruleset.addCombinationDisabledTooltip` | Combination rules cannot be added because an omission rule exists | [source] |
| `serials.publicationPattern.noActivePattern` | Serial has no active publication patterns | [source] |
| `poLine.noneLinked` / `poLine.linkToStart` | No PO line linked / Link a PO line to get started | [source] |
| `serials.notes.isEmptyMessage` | There are no notes associated with this serial | [source] |
| `ruleset.noOmissionRules` / `ruleset.noCombinationRules` | No omission rules for this publication pattern / No combination rules for this publication pattern | [source] |
| `ruleset.noChronologyLabels` / `ruleset.noEnumerationLabels` | No chronology labels for this publication pattern / No enumeration labels for this publication pattern | [source] |
| Backend 405 (delete used pattern via replaceAndDelete) | Cannot delete a serial ruleset which has been used to generate a predicted piece set | mod `SerialRulesetController.groovy` |
| Backend 405 (delete set with receiving pieces) | Cannot delete a predicted piece set which contains receiving pieces | mod `PredictedPieceSetController.groovy` |

## Common Verification Patterns

- Predicted set check (C451603): Start date, Note, Date generated, number of issues = issues per cycle × cycles, each row's publication date and Display summary, "Receiving pieces" column 0 before / 1 after receiving generation.
- Actions menu state check (C451603 steps 12, 16): Generate receiving pieces (active), Delete predicted piece set (active → inactive after receiving pieces exist).
- Receiving check (C451603 steps 17–18): open the POL title in Receiving; one expected piece per non-omitted predicted piece; expected receipt date = publication date + interval days.
