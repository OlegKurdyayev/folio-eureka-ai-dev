---
area: marc-holdings
generated_on: 2026-11-24
previous_version: 2026-11-07
release: Umbrellaleaf (R2 2026) — master; ui-quick-marc 11.1.0 (in progress), mod-quick-marc 8.1.0 (in progress)
generator_version: manual (build-app-context method)
jira_status: ok
---

# FOLIO MARC Holdings (quickMARC) — Business Logic Context

> Built: 2026-11-24 (previous build 2026-11-07) · Release: Umbrellaleaf (R2 2026) master — ui-quick-marc 11.1.0 / mod-quick-marc 8.1.0 in progress; Trillium (R1 2026) = ui-quick-marc 11.0.x / mod-quick-marc 8.0.x; Sunflower (R1 2025) = 10.0.x / 7.0.x (Jira project versions UIQM, MODQM) · Builder: build-app-context method, manual
> Sources: TestRail suite 21 section 16714 "MARC > MARC Holdings" (28 cases) with sub-sections 39766 "Create MARC holdings" (36 cases) and 39767 "Edit MARC holdings" (43 cases) — titles for all, full steps for C350646, C983156, C1045403, C353227, C396394, C496218, C345402; no case created or updated after 2026-11-07.
>          GitHub (HEAD 2026-11-12/16): folio-org/ui-quick-marc (`translations/ui-quick-marc/en.json` = `en_US.json` for every key used here, `package.json`, `src/hooks/useValidation/rules.js`, `src/QuickMarcEditor/utils.js`, `constants.js`, `LeaderField/leaderConfig.js`, CHANGELOG); folio-org/mod-quick-marc (`descriptors/ModuleDescriptor-template.json`, `service/validation/impl/holdings/*`, `convertion/elements/LeaderItem.java`, `convertion/merger/HoldingsRecordMerger.java`, `db/changelog/changes/v2.6.0/field008_holdings.json`, NEWS.md); folio-integration-tests `mod-quick-marc` Karate; stripes-testing `cypress/e2e/marc/marc-holdings`.
>          Jira (anonymous REST): ~60 UIQM/MODQM Done issues mentioning holdings since 2024-09, MODINV/UIIN MARC-holdings bugs since 2025-01, open UIQM/MODQM holdings issues. docs.folio.org quickMARC page (last modified 2026-05-28).
> Related contexts: `marc-bib-quickmarc.md` (shared editor shell, bib-only rules), `inventory.md` (FOLIO holdings record, move, call numbers), `data-import.md` (importing MARC holdings files, section 17338), `consortium-manager.md` (ECS ownership).

## What is MARC Holdings

A holdings record whose source is MARC (`Source` = MARC) is edited in quickMARC, not in the FOLIO holdings form. The MARC holdings record (SRS) drives the FOLIO holdings record: 852 maps to location and call number fields, 004 links to the instance HRID. Some FOLIO-only fields (statistical code, administrative note, ILL policy, receiving history, tags, etc.) are edited in the FOLIO holdings form and must survive a quickMARC save (MODQM-505, MODINV-1323, C983156).

## Entry points

| Action | Where | Result |
|---|---|---|
| `Add MARC holdings record` | Instance detail (Source = MARC) → `Actions` | Pane `New MARC holdings record` (en.json `holdings-record.create.title`; "Create a" prefix removed — UIQM-744), record status `New`, `Save & close` disabled until a change is made (UIQM-522). Not offered on the Central tenant instance page in ECS (C409473) |
| `Edit in quickMARC` | Holdings detail (Source = MARC) → `Actions` | Pane `Edit MARC holdings - Location: {location} - [{callNumber}]` |
| `View source` | Holdings detail → `Actions` | MARC source view; `Edit` and `Print` from the source pane (UIIN-3012, UIQM-352, C543800). `Print` works with view-only permission (C396394). Version history (clock icon) on View source — UIQM-747 (Closed) |
| Import | Data Import, job profile `Default - Create Holdings and SRS MARC Holdings` | Imported MARC holdings record (see `data-import.md`) |

Pane header in edit mode: `Edit MARC holdings - Location: {location} - [{callNumber}]` (en.json `holdings-record.edit.title`). Record status values in the editor header (en.json `record.status.*`): `New`, `Current`, `In progress`, `Error`; `Last updated: {date} {time}` (`record.lastUpdated`).

MARC holdings can be created only for an instance with Source = MARC. Holdings with Source = MARC can be moved to another instance with Source = MARC or FOLIO (C345404, C345405).

## Editor behaviour (shared with bib, holdings-specific values)

- Rows: tag, two indicator boxes, content; row actions `Add a new field` (`record.addField`), `Move field up a row` / `Move field down a row` (`record.moveUpRow` / `record.moveDownRow`), `Delete this field` (`record.deleteField`). Deleted row placeholder: `Field {tag} has been deleted from this MARC record.` with `Undo` (`record.fieldDeleted`, `record.fieldDeleted.undo`; C359241, C359242). Delete confirmation modal `Delete fields` (`record.delete.title`) with `Continue with save` / `Restore deleted field(s)` (C345391, C345392). Deleted fields show a placeholder and can be restored (`Restore deleted field(s)` on the confirmation modal); a restored field returns to the same position (C359178).
- Buttons: `Cancel`, `Save & keep editing` (stays in the editor, keeps focus on the last edited field — UIQM-715), `Save & close`. `Esc` with unsaved changes opens the "Are you sure?" modal (C434155, C434156).
- Empty rows (no tag and no subfield value) are deleted on save (C423484, C423485). No field can be added below `999 ff` (C494043).
- All fields except LDR (and on create also 005, 999) can be moved and saved (C407734, C407738).
- 852 `$b` is filled through `Permanent location look-up` (modal with its own `Save and close`) or by typing a location code (C358990, C569545). Location codes with whitespace are supported (C805766).
- `$` in content is stored as the `{dollar}` code (C451564, C451565); backslash `\` is a valid character and shows in the detail view and browse (MODINV-1285, C877078, C877082).
- Create flow (2026, MODQM-490/491/497, UIQM-806): 003 is removed on save; 035 values are **not** normalized for holdings (C1045403). The edit flow keeps an added 003. Create no longer polls a `status` endpoint — the UI uses the `POST /records-editor/records` response directly (UIQM-806).
- Default fields on create: LDR, 001, 004 (instance HRID, read-only), 005, 008 (holdings defaults), 852 (indicators `\ \`), 999 ff (`src/QuickMarcEditor/constants.js` `CREATE_HOLDINGS_RECORD_DEFAULT_FIELD_TAGS`, `utils.js` `getCreateHoldingsMarcRecordDefaultFields`).
- Read-only tags/content for holdings: 001, 004, 005 (`utils.js` `READ_ONLY_ROWS_FOR_HOLDINGS`). Cannot be deleted: LDR, 001, 003, 004, 005, 008, 852 (`DELETE_EXCEPTION_ROWS`). The 005 **tag box** is editable since UIQM-781, so a 005 can be retagged and then deleted (C916266).
- No `+` on LDR, 001, 005 and (create) 999 ff (`QuickMarcEditorRows/utils.js` `ADD_EXCEPTION_ROWS`; UIQM-671).
- Field protection settings (MARC field protections) do not apply to holdings (`QuickMarcEditorRows.js`: protections only for non-holdings edit).
- 005 is system-managed: updated on every save (C417047) and regenerated (YYYYMMDDHHMMSS) when no 005 is left (C916266, C496218, UIQM-781). An extra 001 is dropped on save (create C387462, edit C387461). An extra 005 is dropped on **create** (C496215), but on **edit** both 005 fields are saved (C496218 step 5).
- Detail view after save: `Record last updated` → `Source` shows the editing user as `<last name>, <first name>`, or last name only when the first name is empty (C356850, C396368).

## Required and non-repeatable fields

| Field | Rule | Message (en.json) |
|---|---|---|
| LDR | Exactly one, 24 characters | `Record cannot be saved. The Leader must contain 24 characters, including null spaces.` |
| LDR positions | Only 05, 06, 17, 18 editable for holdings | `Record cannot be saved. Please check the Leader. Only positions 5, 6, 17, 18 can be edited in the Leader.` (`record.error.leader.forbiddenBytes.holdings`) |
| LDR invalid value | Per-position valid values (see Leader table) | `Record cannot be saved. Please enter a valid {positions}. Valid values are listed at {link}` — link `https://www.loc.gov/marc/holdings/hdleader.html` (`constants.js`; C357572 example `Leader 05, Leader 17 and Leader 18`) |
| Multiple LDR | Blocks save; generic server toast | `Record not saved: Communication problem with server. Please try again.` (C496204, C496206) |
| 001 | One; an added 001 becomes read-only and is dropped on save (C387461, C387462) | `Record cannot be saved. Can only have one MARC 001.` is in the holdings validator list (`rules.js` `BASE_HOLDINGS_VALIDATORS`) but cases show the duplicate dropped — see Known Gaps |
| 004 | Exactly one, read-only, must match an existing instance HRID | `Record cannot be saved. Can only have one MARC 004.` (`record.error.instanceHrid.multiple`; C345399, C503115). API: wrong HRID → 422 `Failed to map marc-holdings record ... No instance found for HRID: <hrid> in tenant: <tenant>`; duplicate → 422 `Is unique tag` (Karate) |
| 008 | Required, one, valid values (32 positions; imported records with fewer positions can still be edited — C387479) | `Record cannot be saved without 008 field` / `Record cannot be saved with more than one 008 field` / `Record cannot be saved. Field 008 contains an invalid value in "{name}" position.` |
| 852 | Required, exactly one, valid location in `$b`, single `$b` with no extra spaces (C386514) | `Record cannot be saved. An 852 is required.` / `Record cannot be saved. Can only have one MARC 852.` / `852 $b contains an invalid Location code. Please try again.` / `<b>{fieldTag}</b> can only have one <b>{subField}</b>.` (`record.error.field.onlyOneSubfield`). API: delete 852 → 422 `Is required tag`, duplicate → 422 `Is unique tag`, FOLIO call number unchanged (Karate) |
| 999 ff | Several allowed on create (C503071) | — |
| Local control fields 002, 009 | Repeatable, with or without `$a` (UIQM-670) | — |
| MARC tag | 3 characters, digits only (MRSPECS-44, C543827, C345393) | `Record cannot be saved. A MARC tag must contain three characters.` / `Invalid MARC tag. Please try again.` (`record.error.tag.length`, `record.error.tag.nonDigits`). Fail/warn summary toast: `Please scroll to view the entire record. Resolve issues as needed and save to revalidate the record.` + `Fail errors: {failCount}` / `Warn errors: {warnCount}` |

Inline errors are rendered with a `Fail: ` prefix in front of the en.json text (C503109, C357572). No Warn toast should appear for holdings — only success or fail (UIQM-740). Error toasts appear before confirmation modals (C375206, C375187). Invalid LDR values in positions 07, 08, 10, 11, 19, 20–23 of an imported record are replaced with valid values on edit (C353984, C358155).

## Leader (holdings) — editable positions and valid values

| Pos | Name (LeaderItem.java) | Valid values | UI default on create |
|---|---|---|---|
| 05 | Holdings record status | `c` Corrected or revised, `d` Deleted, `n` New | — |
| 06 | Holdings type of record | `u` Unknown, `v` Multipart item holdings, `x` Single-part item holdings, `y` Serial item holdings | `u` |
| 17 | Holdings encoding level | `1`, `2`, `3`, `4`, `5`, `m`, `u`, `z` (Holdings level 1–4, level 4 with piece designation, Mixed level, Unknown, Other level) | `u` |
| 18 | Item information | `i` Item information, `n` No item information, blank | `n` |
| 07, 08, 19 | Undefined | `\` only (validated by BE) | — |

Sources: mod-quick-marc `convertion/elements/LeaderItem.java`, `HoldingsLeaderValidationRule.java`; ui-quick-marc `leaderConfig.js`; dropdown labels en.json `leader.*`. Positions 7–16 default `\\\2200000`. MODQM-509 (LDR/05 = `d` sets suppress flags) was verified for MARC bib only — behaviour for holdings not confirmed (Known Gaps).

## 008 (holdings) — positions

From `field008_holdings.json`: `Entered` (00–05, read-only), `AcqStatus` (06), `AcqMethod` (07), `AcqEndDate` (08–11), `Gen ret` (12), `Spec ret` (13–15, array of 3), `Compl` (16), `Copies` (17–19), `Lend` (20), `Repro` (21), `Lang` (22–24), `Sep/comp` (25), `Rept date` (26–31). Box labels = en.json `record.fixedField.<code>`. Defaults on create (`HOLDINGS_FIXED_FIELD_DEFAULT_VALUES`): AcqStatus `0`, AcqMethod `u`, AcqEndDate blanks, Gen ret `0`, Spec ret blanks, Compl `0`, Copies blanks, Lend `u`, Repro `u`, Lang `eng`, Sep/comp `0`, Rept date blanks. docs.folio.org: 008 is edited with dropdowns under each position. MODQM-519 (8.1.0): boxes without allowed values must stay text inputs, not empty dropdowns. Imported records whose 008 is shorter than 32 positions can still be edited (C387479, C387481); blank positions 00–05 must not shift the rest (MODQM-515, 8.0.1).

## Toasts

| Event | Text |
|---|---|
| Create in progress | `Creating record may take several seconds.` |
| Create success | `Record created.` |
| Create failed | `Unable to create record due to system error. Please try again or contact support.` |
| Create slow | `Creating this record is taking longer than expected. Please check back later if record is available or contact support.` |
| Edit saved | `This record has successfully saved and is in process. Changes may not appear immediately.` |
| Record deleted by another user | `This record has been deleted by another user. You can no longer edit this record, hit the cancel button to return to the previous page.` |
| Generic failure | `Record not saved: Communication problem with server. Please try again.` |
| Optimistic locking (banner, not toast) | `This record cannot be saved because it is not the most recent version.` + link `View latest version` with external-link icon; focus moves to banner (C353227 step 10) |

All toast texts above are en.json `record.saveNew.*`, `record.save.success.processing`, `record.save.error.notFound`, `record.save.error.generic`. `Save & keep editing` = `record.save.continue`; `Permanent location look-up` = `permanentLocationLookup`. `Save & close`, `Cancel`, `Close without saving`, `Keep editing` and the "Are you sure?" modal are not in quickMARC en.json (stripes/ui-inventory) — quote them from cases (C356843, C434155, C434156). UIQM-825 (11.1.0): the deleted-by-another-user toast must be the `record.save.error.notFound` text.

## Mapping to the FOLIO holdings record

- 852 → permanent location, call number, prefix, suffix, call number type (from the 852 first indicator — MODQM-506, C389500, C389503). Deleting the mapped 852 subfields clears the FOLIO call number fields (C375188).
- Changing holdings call number components updates the item's effective call number metadata; changing unrelated fields does not (C388510, C388511).
- FOLIO-only fields and tags are preserved after a quickMARC save — regression set from MODQM-505, MODQM-513 (number of items, receipt status — still open), MODINV-1323 (C983156). Preserved list in code (`HoldingsRecordMerger.java` ignore list): discoverySuppress, statisticalCodeIds, administrativeNotes, additionalCallNumbers, illPolicyId, digitizationPolicy, retentionPolicy, acquisitionFormat, acquisitionMethod, receivingHistory, temporaryLocationId, tags. `numberOfItems` and `receiptStatus` are **not** in the list — matches open MODQM-513.
- Other mapped fields are set to null when absent from the MARC (MODQM-506 "Set Holdings fields to null in mergers") — that is why deleting 852 subfields clears call number parts (C375188).
- ECS: after `Update ownership` to another member tenant, the 852 `$b` location code must change to the new location (MODINV-1389, UIQM-834 — fixed for Umbrellaleaf/snapshot, not backported to Trillium; before the fix any later quickMARC save failed with the invalid-location error). Ownership update re-creates the SRS MARC holdings in the target tenant and marks the source-tenant SRS record DELETED; quickMARC must be available on the moved record (MODINV-1266, C788750). Create/edit in ECS tenants regressed in Sunflower (MODINV-1327, MODINV-1330).

## ECS / Multi-Tenant Notes

- Create/edit MARC holdings on Shared and Local MARC bib from a Member tenant is supported (UIQM-771, MODQM-465, MODINV-1240; C784586, C788740, C788741, C788743). Delete on Shared/Local bib from Member: C788748, C788749.
- Central tenant instance page has no `Add MARC holdings record` action (C409473).
- View source label shows `Local MARC holdings record` for a local record instead of `Shared MARC holdings record` (UIIN-3433 requirement text; string lives in ui-inventory, not verified in en.json).
- Comment on MODINV-1389: no ECS library uses MARC holdings in production — ECS holdings cases are Extended priority.

## Delete

- Delete is done from the FOLIO holdings detail (`Actions` → `Delete`), not in quickMARC. A holdings record with items cannot be deleted (C345402); without items it is deleted (C345403).
- With limited permissions (Inventory manage + Quick-Marc-Holdings-Editor create/manage) the SRS `DELETE /source-storage/records/<id>` must return 204, not 403 (UIIN-3434, ui-inventory 14.0.0). Deleted holdings may linger in the instance `Holdings` accordion until refresh because of search indexing (UIIN-3498, fixed 14.0.0).

## API surface (mod-quick-marc, interface `marc-records-editor` 7.0)

| Method / path | permissionsRequired | Notes |
|---|---|---|
| `GET /records-editor/records?externalId=<holdingsId>` | `marc-records-editor.item.get` | Load MARC holdings for editing |
| `POST /records-editor/records` | `marc-records-editor.item.post` | Create (`_actionType: create`), 201 |
| `PUT /records-editor/records/{parsedRecordId}` | `marc-records-editor.item.put` | Edit (`_actionType: edit`), 202; 422 on validation (`Is unique tag`, `Is required tag`) |
| `POST /records-editor/validate` | `marc-records-editor.validate.post` | Spec-based validation (fail/warn) |
| `GET /marc-specifications/holdings/008` | `marc-specifications.item.get` | 008 spec for the editor |

Source: `descriptors/ModuleDescriptor-template.json`. The `status` endpoint is gone from the descriptor (UIQM-806/MODQM-491), yet `ui-quick-marc.quick-marc-holdings-editor.create` still lists `marc-records-editor.status.item.get` (package.json) — see Known Gaps. Optimistic locking is based on the SRS `generation` field (MODQM-478, 8.0.0; UIQM-791 verified for holdings). Downstream checks in Karate: `GET /holdings-storage/holdings/{id}`, `GET /source-storage/source-records`.

## Permissions (Okapi → Eureka, from team cases)

| Okapi permission | Eureka capability (type – resource – action) |
|---|---|
| `quickMARC: View MARC holdings record` | `data - UI-Quick-Marc Quick-Marc-Holdings-Editor - view` (docs.folio.org quickMARC permissions table; not yet in a case) |
| `quickMARC: View, edit MARC holdings record` | `data - UI-Quick-Marc Quick-Marc-Holdings-Editor - manage` |
| `quickMARC: Create a new MARC holdings record` | `data - UI-Quick-Marc Quick-Marc-Holdings-Editor - create` |
| `Inventory: View instances, holdings, and items` | `data - UI-Inventory Instance - view` |
| `Inventory: View, create, edit holdings` | `data - UI-Inventory Holdings - edit` |
| `Inventory: All permissions` | `data - UI-Inventory - manage` |

Recent cases use the two-column Okapi / Eureka table in preconditions (C1045403) or the Eureka list only (C983156). Create/manage capability names also appear in UIIN-3434 and UIIN-3498 preconditions.

UI permission sets (`package.json`): view = `marc-records-editor.item.get`, `inventory.instances.item.get`, `instance-authority.linking-rules.collection.get`; all (view, edit) = view + `marc-records-editor.item.put`, `marc-specifications.item.get`; create = view + `marc-records-editor.status.item.get`, `marc-records-editor.item.post`, `marc-specifications.item.get`. Note: neither holdings set includes `marc-records-editor.validate.post` (bib/authority sets do).

## Existing automated coverage

- Karate: folio-integration-tests `mod-quick-marc/src/main/resources/promin/mod-quick-marc/features/quick-marc-holdings-records.feature` (runner `QuickMarcApiTest.testQuickMarcHoldingsRecordsFeature`): valid 004, 008 present, valid 852 location, create via quickMARC, edit tags, remove optional tag, add tag (SRS updated), update twice, invalid 004 HRID → 422, duplicate 004 → 422, duplicate 852 → 422, delete 852 → 422. Also `quick-marc-handle-literal-dollar.feature`, `quick-marc-tags-order.feature` (shared).
- Cypress: stripes-testing `cypress/e2e/marc/marc-holdings/` (81 specs, ~95 C-ids incl. all create/edit validations, LDR, 008, 001/004/005/852 duplicates, optimistic locking, Save & keep editing, call number mapping, backslash, `{dollar}`, 003/035), `cypress/e2e/consortia/marc/marc-holdings/` (create/edit/delete on shared/local bib), `cypress/e2e/consortia/inventory/holdings/update-ownership-of-marc-holdings-record-which-belongs-to-shared-marc-bib.cy.js` (C788750), `cypress/e2e/inventory/keyboard-shortcut/keyboard-shortcuts-edit-marc-holdings*.cy.js`, `cypress/e2e/data-import/importing-marc-holdings-files/`. Fragment: `cypress/support/fragments/quickMarcEditor.js`.
- Before writing a case, check the C-id list above — most single-rule validations already exist as one create + one edit case.

## Key Business Rules for Test Cases

1. MARC holdings exist only under an instance with Source = MARC; the create action is absent for FOLIO-source instances (cases).
2. 852 is required and non-repeatable; 004 and 008 are non-repeatable; 008 is required (en.json + cases).
3. Only LDR positions 05, 06, 17, 18 are editable — do not reuse the bib list (5, 6, 7, 8, 17, 18, 19) or the authority list (5, 17, 18) (en.json).
4. An added duplicate 001 is dropped on save (C387461, C387462); an added duplicate 005 is dropped on create (C496215) but both 005 are saved on edit (C496218); duplicate 004, 008, 852 or LDR block the save (C345399, C503115, C499629, C499630, C350721, C503109, C496204, C496206).
5. Every save updates 005 and the `Record last updated` source user (C417047, C356850, C353227).
6. A quickMARC save must not clear FOLIO-only holdings fields or tags (MODQM-505, MODINV-1323).
7. Call number type follows the 852 first indicator, including after edit (MODQM-506).
8. Optimistic locking: two users editing the same MARC holdings record — the second save shows the conflict banner; `Save & keep editing` follows the same rule (C353227, C353229, C360103).
9. Create and edit are separate sections (39766, 39767); the team writes the same validation as two cases, one per flow. Keep that split.
10. 001, 004 and 005 rows are read-only; LDR, 001, 003, 004, 005, 008 and 852 cannot be deleted from the row action (ui-quick-marc `src/QuickMarcEditor/utils.js`; C375123 for 852).
11. The 005 tag box is editable (like 008); a record saved without any 005 gets a system-generated 005 (UIQM-781, C916266, C496218).
12. 004 is pre-filled with the instance HRID on create and must match an existing instance HRID; otherwise the API returns 422 `No instance found for HRID` (`getCreateHoldingsMarcRecordDefaultFields`; Karate `quick-marc-holdings-records.feature`).
13. 852 `$b` is non-repeatable within the field and must be a valid location code of the current tenant (`rules.js` `NON_REPEATABLE_SUBFIELD`, `VALID_LOCATION`; C386514, C805766).
14. LDR 05 accepts `c`, `d`, `n`; 06 `u`, `v`, `x`, `y`; 17 `1`–`5`, `m`, `u`, `z`; 18 `i`, `n`, blank; 07, 08, 19 must be `\` (mod-quick-marc `LeaderItem.java`; C357571–C357573, C357063, C357572).
15. Invalid LDR 07, 08, 10, 11, 19, 20–23 in an imported record are auto-corrected when edited (C353984, C358155).
16. Validation error text in the editor is prefixed `Fail: `; no Warn toast for holdings (C503109, UIQM-740).
17. Fields with no tag and no subfield content are removed on save (C423484, C423485); no field can be added below `999 ff` on create (C494043, UIQM-671).
18. All fields except LDR can be reordered on edit; on create LDR, 001, 005, 999 ff are fixed (C407734, C407738, `QuickMarcEditorRows/utils.js` `MOVE_EXCEPTION_ROWS_FOR_CREATE_DERIVE`).
19. Deleting fields shows the `Delete fields` modal on save; `Restore deleted field(s)` puts them back in the same position (C345392, C359178).
20. `Esc`, `Cancel` or `X` with unsaved changes opens the "Are you sure?" modal, also when fail errors are shown (C434155, C434156, UIQM-743).
21. `Save & keep editing` keeps the pane open, saves the changes and keeps focus on the last edited field (C360100, C360101, C605893, UIQM-715).
22. Optimistic locking is triggered by the SRS generation, so a change to any field — mapped or not — by another user blocks the second save with the banner (MODQM-478, UIQM-791, C353227).
23. FOLIO-only holdings fields in the `HoldingsRecordMerger` ignore list survive a quickMARC save; Number of items and Receipt status are still cleared (MODQM-505, MODINV-1323, open MODQM-513; C983156).
24. Deleting the 852 call-number subfields clears the FOLIO call number fields (MODQM-506, C375188); changing call number parts updates item effective call number (C388510).
25. 003 is removed and 035 is not normalized when creating MARC holdings; 003 is kept on edit (MODQM-497, C1045403).
26. `$` is stored and shown as `{dollar}`; `\` is kept as a literal backslash (MODQM-256, MODINV-1285; C451564, C451565, C877078).
27. Local control fields 002 and 009 are repeatable with or without `$a` (UIQM-670; C503164–C503166).
28. A holdings record with items cannot be deleted; deletion is from the FOLIO holdings `Actions` menu (C345402, C345403).
29. Holdings with Source = MARC can be moved to another MARC or FOLIO instance (C345404, C345405).
30. ECS: the Central tenant has no `Add MARC holdings record`; member tenants can create/edit on shared and local bibs; after `Update ownership` the 852 `$b` follows the new location and quickMARC stays available (C409473, C784586, C788750, MODINV-1266, MODINV-1389).
31. `quickMARC: View MARC holdings record` (view capability) is enough to open View source and `Print` (C396394).
32. quickMARC validation applies only to UI create/edit, not to Data Import (docs.folio.org quickMARC page).

## Authoring style (measured 2026-11-07)

Type `Functional` in the create/edit sub-sections (type_id 6 in all 82 cases); the parent section 16714 has older cases of another type. Priority Medium dominates. Full-case sample (C350646, C983156, C1045403): 14, 7 and 8 steps; Test Group Critical Path for the core create journey, Extended for regression bugs. Step count was not measured across the section. Titles name the record in quotes: `"MARC Holdings"` / `"MARC holdings"`, often with the flow: `... when creating "MARC Holdings" record`. Data setup is a Data Import file with the default job profile, referenced in preconditions.

## Known Gaps / Items to Verify

- [ ] Eureka capability for view-only (`Quick-Marc-Holdings-Editor - view`) — confirmed by docs.folio.org only, not yet in a case.
- [ ] MODQM-513 (number of items, receipt status cleared after quickMARC save) is still In Refinement — expect failure until fixed; code confirms (`HoldingsRecordMerger` ignore list lacks both fields).
- [ ] 008 control type: docs.folio.org says dropdowns per position; MODQM-519 (fixed in mod-quick-marc 8.1.0) restored text inputs for boxes without allowed values. Check which boxes are dropdowns on the target build before writing 008 steps.
- [ ] Duplicate 001: `rules.js` has a NON_REPEATABLE 001 rule with `Can only have one MARC 001.`, while C387461/C387462 expect the added 001 to be read-only and silently dropped. Follow the cases; verify in env.
- [ ] Duplicate 005 on edit: C496218 expects both 005 fields to be saved; UIQM-781 asks for a non-repeatable error when several 005 have data, and its comment says a holdings story for that message is still to be written. No holdings 005 rule in `rules.js`.
- [ ] docs.folio.org says 001, 004, 005 and 999ff "cannot be edited or deleted"; since UIQM-781 the 005 tag can be edited and the field deleted (C916266). Docs are outdated on 005.
- [ ] `ui-quick-marc.quick-marc-holdings-editor.create` still requires `marc-records-editor.status.item.get`, which mod-quick-marc 7.0 no longer provides (UIQM-806). Holdings permission sets also lack `marc-records-editor.validate.post`. Possible Eureka capability mismatch — check if a create-only user can save.
- [ ] MODQM-509 (LDR/05 = `d` sets staff suppress / suppress from discovery) was verified on MARC bib; effect for MARC holdings (holdings have `discoverySuppress` only) not confirmed.
- [ ] UIQM-834 / MODINV-1389 fix is not in Trillium — on Trillium builds the 852 `$b` keeps the old tenant location after ownership update.
- [ ] Version history on MARC holdings View source (UIQM-747) — strings (`Version history`, `Changed`, `Current version`, `Original Version`) are from the story, not from en.json (they belong to ui-inventory); verify before asserting.
- [ ] Optimistic locking banner, "Are you sure?" modal, `Save & close`, `Close without saving` — texts come from cases (C353227, C434155) and stripes-testing; not in quickMARC en.json.
- [ ] UIQM-710 (Open): inline format of 007/008 error messages for MARC holdings — 008 messages may change.
