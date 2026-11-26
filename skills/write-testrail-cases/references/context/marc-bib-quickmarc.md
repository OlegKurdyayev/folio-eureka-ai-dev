# FOLIO quickMARC — MARC Bibliographic Editing — Business Logic Context

> Built: 2026-11-25 · Release: Umbrellaleaf R2 2026 (master: ui-quick-marc 11.1.0, mod-quick-marc 8.1.0, mod-record-specifications 3.1.0 — all unreleased "next minor"; Trillium R1 2026 = ui-quick-marc 11.0.x / mod-quick-marc 8.0.x) · Builder: build-app-context (full refresh; previous build 2026-07-21, delta 2026-11-07 folded into the main sections)

> Sources (2026-11-24): TestRail suite 21 section 1285 "MARC" > 16715 "MARC Bibliographic" (23705 Create new MARC bib, 23706 Derive MARC bib, 24360 Edit MARC bib — each with Manual linking / Automated linking / Consortia sub-sections — plus 27186 Optimistic locking, 39274 "API | MARC validation rules (bibliographic)", 58862 Version history). GitHub: folio-org/ui-quick-marc (translations/ui-quick-marc/en.json, package.json, src/hooks/useValidation, src/QuickMarcEditor), folio-org/mod-quick-marc (descriptors/ModuleDescriptor-template.json, docs/features/*.md, NEWS.md), folio-org/mod-record-specifications (translations/mod-record-specifications/en.json, validator severities, descriptor), folio-org/stripes-components (en.json), folio-org/folio-integration-tests (mod-quick-marc), folio-org/stripes-testing (cypress/e2e/marc/marc-bibliographic), folio-org/docs (Metadata/Inventory/QuickMARC.md). Jira: UIQM + MODQM Done stories/bugs resolved 2024-06-01 → 2026-11-24 (105 issues; fixVersions are module versions, so the window is by resolution date). Confluence: FOLIJET "MARC validation - Logic" (page 1407631).
> This file covers **MARC Bibliographic** editing only. MARC Holdings (section 16714 → `marc-holdings.md`) and MARC Authority (section 17151 → `marc-authority.md`) share the quickMARC editor shell and part of the string vocabulary but have their own field rules — don't assume a Bib rule (e.g. 1XX/245/010 handling) transfers unchanged.
> String tags: **[en.json]** = `folio-org/ui-quick-marc/translations/ui-quick-marc/en.json` key named; **[rspec]** = `folio-org/mod-record-specifications/translations/mod-record-specifications/en.json` (backend validation message, shown after the UI's `Fail:`/`Warn:` prefix); **[C…]** = confirmed in a TestRail case. `en_US.json` matches `en.json` except it lacks the 3 newest 008 labels (`record.fixedField.Activity report` / `Serial zine` / `Serial podcast`).

---

## What is quickMARC (Bibliographic)

quickMARC is the in-browser MARC editor FOLIO uses for creating, deriving, and editing MARC Bibliographic records directly (tag/indicator/subfield grid), as an alternative to Data Import. It is a plugin (`stripes.actsAs: ["plugin"]`, ui-quick-marc package.json), not an app: it is opened from the **Inventory** app (`Actions > New MARC bibliographic record`, `Actions > Derive new MARC bibliographic record`, `Actions > Edit MARC bibliographic record`) and saves through mod-quick-marc into SRS **and** the Inventory Instance. quickMARC renders the fixed fields (**LDR** and **008**, plus **006**/**007**) as structured dropdowns/boxes rather than raw character positions, validates on save (front-end rules + backend MARC-specification rules), and supports linking bib access points to MARC Authority records.

---

## Key Terms

| Term | Definition |
|---|---|
| LDR (Leader) | 24 fixed character positions; quickMARC splits some into read-only groups and some into individual dropdowns/boxes (Status, Type, BLvl, Ctrl, ELvl, Desc, MultiLvl) |
| 008 field | Fixed-length control field (normalised to 40 chars for bib — mod-quick-marc docs/features/get-record.md); positions render as labeled dropdowns/boxes, several mandatory |
| Controlled field | A bib field linked to and governed by a MARC Authority heading — the controlled subfields and `$0` become read-only |
| Manual linking | Staff links one bib field to an authority record via the field's link icon and the **Select MARC authority** plugin |
| Automated linking | `Link headings` button links every eligible field whose `$0` matches an authority `naturalId` (rule must have `autoLinkingEnabled: true`) |
| Derive | Create a new bib record starting from a copy of an existing one |
| Optimistic locking | Save-conflict protection based on the SRS record `generation` (`sourceVersion`) since Trillium (MODQM-478, UIQM-791) |
| MARC validation rules / MARC specification | Tenant-level field/indicator/subfield rules stored in mod-record-specifications (`specification-storage`); quickMARC validates against them on save |
| Fail / Warn | Validation issue severity: Fail (`ERROR`) blocks save, Warn (`WARN`) lets the user save again as-is |
| 999 ff | System-managed field carrying `$i` (Instance UUID) and `$s` (SRS id) |

---

## UI structure [confirmed]

- Entry points from Inventory: `Actions > New MARC bibliographic record` (create), `Actions > Derive new MARC bibliographic record` (derive), `Actions > Edit MARC bibliographic record` (edit), `Actions > View source` (read-only MARC view). (docs QuickMARC.md, C353225, C514849)
- Pane titles [en.json]: create `bibliographic-record.create.title` = `New {shared/local} MARC bib record`; edit `bibliographic-record.edit.title` = `Edit {shared/local} MARC record - {title}`; derive `bibliographic-record.derive.title` = `Derive a new {shared/local} MARC bib record`. The `shared`/`local` word only renders in ECS; non-ECS shows e.g. `New MARC bib record` (UIQM-744 removed the old "Create a" wording — older cases saying `Create a new MARC bib record` are outdated). ECS example: `Derive a new local MARC bib record` (C514879).
- Pane header shows `Status:` (`New` / `Current` / `In progress` / `Error`) and `Last updated: {date} {time}` [en.json `record.status*`, `record.lastUpdated`].
- Each field row: tag box, two indicator boxes, content box; row icons [en.json]: `Move field up a row`, `Move field down a row`, `Add a new field`, `Delete this field`; protected fields show `This field is protected.` (`record.protectedField`). 00X fields can be moved (UIQM-563); LDR, 005 and the system `999 ff` row cannot be moved and the system `999 ff` row has no `+` (UIQM-671; C407733/C407736 titles).
- LDR: read-only grouped boxes (00-04, 09-16, 20-23) plus dropdowns for the editable positions `Status`(05), `Type`*(06), `BLvl`*(07), `Ctrl`(08), `ELvl`(17), `Desc`(18), `MultiLvl`(19); on Create, `Type` and `BLvl` always carry the required asterisk (UIQM-666); hovering a box shows its full name.
- 008 renders only after LDR Type + BLvl are chosen; positions needing a valid selection are **highlighted in red** until fixed (docs QuickMARC.md, C514849 step 2-3). Text-input 008 boxes (e.g. Date 1/Date 2) stay text inputs, not dropdowns (MODQM-519 regression fixed).
- 010 rows on Create/Edit/Derive show a search icon with tooltip `Search for records by 010 value(s)` [en.json `record.searchLink`] that opens Inventory advanced search (LCCN, normalized; exact phrase; up to six 010s OR-ed) in a new tab (UIQM-660/672).
- `Link headings` button in the pane header with an info-tip popover [en.json `autoLinkingButton.infotip`]: `The Link headings button links all access point bibliographic fields that contain a $0 to authority heading. If a field does not have a $0, you can manually link the field to an authority heading by selecting the Link button on the field itself.` + `Learn more` link (UIQM-807; C422143, C1045434, C1045435). The button is **not displayed** when auto-linking is disabled for all heading types (C389481/C389482/C389484).
- Save buttons: `Save & close`, `Save & keep editing` [en.json `record.save.continue`]. After `Save & keep editing` focus returns to the last focused field/button, including after a modal closes (UIQM-715, UIQM-728).
- Delete-fields confirm flow [en.json `record.delete.*`]: modal `Delete fields`, body `By selecting <b>Continue with save</b>, then <b>{count} field(s)</b> will be deleted and this record will be updated. Are you sure you want to continue?`, buttons `Continue with save` / `Restore deleted field(s)`.
- Validation wait modal (shown if MARC validation takes > 2 s — UIQM-632) [en.json]: heading `MARC validation rules check`, body `There might be a slight delay while the system is checking against the library's validation rules. You can continue to work with {appName} in a different browser tab if needed.`
- `Cancel` / `X` / `Esc` with unsaved changes shows the unsaved-changes modal even when Warn/Fail errors are displayed (UIQM-743).

---

## API surface

### mod-quick-marc (`marc-records-editor` 7.0, `marc-specifications` 1.2 — descriptors/ModuleDescriptor-template.json)

| Method | Path | permissionsRequired | Behaviour (mod-quick-marc docs/features) |
|---|---|---|---|
| GET | `/records-editor/records?externalId={instanceId}` | `marc-records-editor.item.get` | quickMARC view incl. `sourceVersion` (SRS generation), link details (bib only), `isProtected` per field, `updateInfo.updatedBy`. 400 bad UUID, 404 no SRS record |
| POST | `/records-editor/records` | `marc-records-editor.item.post` | Create (also used by Derive). Validates first (bib: `001 MISSING_FIELD` skipped); creates Instance, then SRS snapshot + record (`generation 0`); adds `001` (HRID) and `999 ff $i`; removes `003`, normalises `035`. 201 / 422 `validationResult` (nothing persisted) |
| PUT | `/records-editor/records/{parsedRecordId}` | `marc-records-editor.item.put` | Update; path id must equal body `parsedRecordId` (400). 202 empty body; 409 on generation mismatch (optimistic locking); 422 validation |
| POST | `/records-editor/validate` | `marc-records-editor.validate.post` | Always 200 with `validationResult.issues` (tag, severity, definitionType, helpUrl, message); spec resolved by `marcFormat` |
| POST | `/records-editor/links/suggestion` | `marc-records-editor.links.suggestion.post` | `Link headings`; `authoritySearchParameter` = `NATURAL_ID` (default) or `ID`; `ignoreAutoLinkingEnabled` default false |
| GET | `/marc-specifications/{recordType}/{fieldTag}` | `marc-specifications.item.get` | Fixed-field (LDR/006/007/008) position specs and `allowedValues` |

Related: mod-record-specifications `specification-storage` (`GET /specification-storage/specifications`, `.../{id}/rules`, `PATCH .../rules/{ruleId}`, `.../{id}/fields`, `POST .../{id}/sync` — "restore from LOC", with `preserveLocal` param; permissions `specification-storage.*`); mod-entities-links `GET/PATCH /linking-rules/instance-authority[/{id}]`, `GET/PUT /links/instances/{instanceId}`; mod-settings `GET/POST/PUT /settings/entries` for the LCCN duplicate-check setting (C514849 preconditions).

---

## Exact UI Texts

### Backend validation messages (MARC specification rules) [rspec] — rendered inline as `Fail: <msg>` or `Warn: <msg>`

| Rule code | Message | Severity (validator class) |
|---|---|---|
| missingField | `Field {tag} is required.` (C503056, C552462 `Field 700 is required.`; Cypress `calloutNo245MarcTag`) | Fail |
| nonRepeatableField | `Field is non-repeatable.` (C380644, C496209 for 005) | Fail |
| nonRepeatable1XXField | `Field 1XX is non-repeatable.` | Fail |
| nonRepeatableRequired1XXField | `Field 1XX is non-repeatable and required.` (Cypress quickMarcEditor.js) | Fail |
| invalidFieldTag | `Tag must contain three characters and can only accept numbers 0-9.` | Fail |
| invalidIndicator | `Indicator must contain one character and can only accept numbers 0-9, letters a-z or a '#'.` | Fail |
| undefinedIndicatorCode | `{order} Indicator '{code}' is undefined.` (blank shown as `\` — MODQM-448) | **Warn** |
| missingSubfield | `Subfield '{code}' is required.` | Fail |
| nonRepeatableSubfield | `Subfield '{code}' is non-repeatable.` | Fail |
| undefinedSubfield | `Subfield '{code}' is undefined.` | **Warn** |
| undefinedField | `Field is undefined.` | **Warn** (default severity) |
| invalidLccnSubfieldValue | `010 $a is in an invalid format.` (MODQM-447) | Fail |

Severities from mod-record-specifications-validator `*RuleValidator.severity()` (default WARN; required/non-repeatable/invalid → ERROR). Matches Confluence "MARC validation - Logic": undefined indicator/subfield codes and fields not on the validation list → Warn; required / NR violations → prevent save. Each inline issue may carry a `Help` link [en.json `record.errorMessage.help`] to the field's help URL (UIQM-656).

### Front-end validation messages for MARC bib [en.json, referenced in src/hooks/useValidation/rules.js `BASE_BIB_VALIDATORS`]

| Condition | Key | Message |
|---|---|---|
| Non-editable LDR position changed | `record.error.leader.forbiddenBytes.bibliographic` | `Record cannot be saved. Please check the Leader. Only positions 5, 6, 7, 8, 17, 18 and/or 19 can be edited in the Leader.` |
| Invalid LDR position value | `record.error.leader.invalidPositionValue` | `Record cannot be saved. Please enter a valid {positions}. Valid values are listed at {link}` (e.g. `...valid Leader 06 and Leader 07. Valid values are listed at https://loc.gov/marc/bibliographic/bdleader.html` — UIQM-725) |
| Invalid 008 dropdown value (one error per position, all shown together — UIQM-711) | `record.error.008.invalidValue` | `Record cannot be saved. Field 008 contains an invalid value in "{name}" position.` |
| Wrong length in 006/007/008 text box | `record.error.fixedField.invalidLength` | `Invalid {name} field length, must be {length} characters.` (C503013 `Invalid Ctry field length, must be 3 characters.`; UIQM-698 `Invalid Proj field length, must be 2 characters`) |
| `$9` in a linkable field | `record.error.$9` | `$9 is an invalid subfield for linkable bibliographic fields.` |
| Controlled subfield entered in an uncontrolled box | `record.error.fieldIsControlled` | `A subfield(s) cannot be updated because it is controlled by an authority heading.` (shown only under the offending linked field — UIQM-694) |
| Duplicate LCCN (setting enabled) | `record.error.010.lccnDuplicated` | `010 $a already exists.` (C514849, C514879) |

### Save-flow callouts [en.json; usage in src/QuickMarcEditor/useSaveRecord/useSubmitRecord/useSumbitRecord.js]

| Event | Message |
|---|---|
| Edit saved (incl. shared/ECS) | `This record has successfully saved and is in process. Changes may not appear immediately.` (`record.save.success.processing`; C405507; Cypress `calloutUpdatedRecord`) |
| Create | First `This record has successfully saved and is in process. Changes may not appear immediately.`, then `Record created.` (`record.saveNew.success`) |
| Derive | First `Creating record may take several seconds.` (`record.saveNew.onSave`), then `Record created.` (Cypress `calloutOnDeriveFirst/Second`) |
| Warn-only errors on save | `<b>Please scroll to view the entire record. Resolve issues as needed and save to revalidate the record.</b>{breakingLine}Warn errors: <b>{warnCount}</b>` (orange toast — UIQM-730) |
| Fail errors on save | Same lead + `Fail errors: <b>{failCount}</b>{breakingLine}Record cannot be saved with a fail error.`; with both: `record.save.error.failAndWarn` lists Warn then Fail counts (red toast) |
| Optimistic-locking conflict (HTTP 409) | Conflict banner under the pane header (stripes-components `optimisticLocking.saveError`): `This record cannot be saved because it is not the most recent version.` + link `View latest version` (`optimisticLocking.latestVersion`), focus moves to the banner (C353225; Cypress quickMarcEditor.js) |
| Record deleted by another user (`NOT_FOUND`) | `This record has been deleted by another user. You can no longer edit this record, hit the cancel button to return to the previous page.` (`record.save.error.notFound`) |
| Fixed-field length rejected by backend | `Record not saved. Please check the character length of the fixed fields.` (`ILLEGAL_FIXED_LENGTH_CONTROL_FIELD`) |
| Any other HTTP error | `Record not saved: Communication problem with server. Please try again.` (`record.save.error.generic`; C496193 shows it can surface for a malformed record) |
| Link headings — linked | `Field <b>{fieldTags}</b> ... has been <b>linked</b> to MARC authority record(s).` (`records.autoLink.linkedFields`; C388560) |
| Link headings — no match | `Field <b>{fieldTags}</b> ... must be set manually by selecting the link icon.` (`records.autoLink.notLinkedFields`); several candidates: `... There are multiple authority records that can be matched to this bibliographic field.` (`.multiple`) |
| Link headings — system error | `Unable to link record(s) due to system error. Please try again or contact support.` |
| Manual link success | `Field <b>{tag}</b> has been <b>linked</b> to a MARC authority record.` (`record.link.success`; C365134) |
| Invalid heading picked in plugin | `You have selected an invalid heading based on the bibliographic field you want controlled. Please revise your selection.` |
| Unlink confirm | Modal `Unlink from MARC authority record`, body `By selecting <b>Unlink</b>, then field <b>{tag}</b> will be unlinked from the MARC authority record. Are you sure you want to continue?`, buttons `Cancel` / `Unlink` |

> **Legacy keys — do not assert for MARC bib.** These are still in `en.json` but are not referenced anywhere in `ui-quick-marc/src` (bib validation of these conditions moved to the backend spec rules above): `record.error.heading.empty` / `.multiple`, `record.error.title.empty` / `.multiple`, `record.error.010.multiple`, `record.error.leader.fixedFieldMismatch`, `record.error.bib.leader.invalid005PositionValue`, `record.error.subfield`, `record.save.error.derive`, `record.saveNew.delay`, `record.fieldDeleted`, `remove-authority-linking.modal.*`, `record.error.leader.initial.invalidPositionValue`. Cypress fragment constants still hold some of them (e.g. `Record cannot be saved with more than one field 245.`) — treat as stale. Live cases already assert the backend wording instead (C380644 `Fail: Field is non-repeatable.`, C503056 `Field 245 is required.`).

> **Warn vs Fail matters for case design:** a Warn-only result lets the user save again as-is (team convention: "ignore errors and Save again" — C496208/C496193); a Fail hard-blocks save. Don't write a negative case that treats every inline error as save-blocking.

---

## Settings and configuration

| Setting | Where | Notes |
|---|---|---|
| Duplicate LCCN (010 $a) check | mod-settings entry `scope: ui-quick-marc.lccn-duplicate-check.manage`, `key: lccn-duplicate-check`, `value: {"duplicateLccnCheckingEnabled": true}` via `/settings/entries` — **no UI** (UIQM-678, UIQM-693; C514849 preconditions) | Per tenant; permission `ui-quick-marc.settings.lccn-duplicate-check.edit` (package.json, not visible) |
| MARC validation rules (bib specification) | API only — mod-record-specifications `specification-storage` (TestRail 39274 "API \| MARC validation rules (bibliographic)" and children 39580/39275/39276/39356; Cypress `validation-rules/`) | Tenant cannot change rules for LDR, 001, 005, 245, 999 ff, 006/007/008 except Help URL (Confluence 1407631). `POST .../sync` restores System/Standard rules from LOC (C1563888, MRSPECS-87) |
| Linking rules (auto-linking on/off per field) | mod-entities-links `/linking-rules/instance-authority` (docs QuickMARC.md) | Disabling all heading types hides `Link headings` (C389484) |
| Version history card count | Settings > Inventory > Version history (TestRail 57970/57971) | Central-tenant value applies to Shared Instances with source MARC (C1621424) |
| Field protection | Data import field-protection settings (`/field-protection-settings/marc`) | Protected fields flagged `isProtected` in GET response (get-record.md) |

---

## Capability Sets (Eureka) — dual-column convention confirmed live in corpus

Real cases (C496193, C503013, C496208, C514879) use the team's two-column Okapi/Eureka table:

| Permission name for Okapi env. | Capabilities/Sets for Eureka env. |
|---|---|
| Inventory: All permissions | `Data - UI-Inventory - Manage` (C496208; docs Inventory/_index.md; Cypress capabilitySets `uiInventory`) |
| Inventory: View instances, holdings, and items | `Data - UI-Inventory Instance - View` (C496210, C514879; docs Inventory/_index.md; Cypress `uiInventoryInstanceView`) |
| quickMARC: View MARC bibliographic record | `Data - UI-Quick-Marc Quick-Marc-Editor - View` (docs QuickMARC.md; Cypress `uiQuickMarcQuickMarcEditorView`) |
| quickMARC: Create a new MARC bibliographic record | `Data - UI-Quick-Marc Quick-Marc-Editor - Create` (C496208; docs QuickMARC.md; no bib `Create` entry in Cypress capabilitySets) |
| quickMARC: View, edit MARC bibliographic record | `Data - UI-Quick-Marc Quick-Marc-Editor - Manage` (C496208, C496210; Cypress `uiQuickMarcQuickMarcEditor`) |
| quickMARC: Derive new MARC bibliographic record | `Procedural - UI-Quick-Marc Quick-Marc-Editor Derive - Execute` (C496210, C514879; Cypress `uiQuickMarcEditorDuplicate`) |
| quickMARC: Can Link/unlink authority records to bib records | `Procedural - UI-Quick-Marc Quick-Marc-Authority-Records Link-Unlink - Execute` (docs QuickMARC.md; Cypress `uiQuickMarcQuickMarcAuthorityRecordsLinkUnlink`) |

All names above are confirmed by the Cypress dictionary, docs, and/or case preconditions (the corpus writes the type/action in lower case, `data - ... - manage`; use the title-case form here). No `(derived)`/unconfirmed capability names remain in this file.

Legacy permission ids (ui-quick-marc package.json): `ui-quick-marc.quick-marc-editor.view` / `.create` / `.all` / `.derive.execute`, `ui-quick-marc.quick-marc-authority-records.link-unlink.execute`. Notable sub-permissions: `.view` includes `audit.config.groups.settings.*` (Version history, UIQM-761 era); `.create`, `.all` and `.derive.execute` include `marc-records-editor.validate.post`, `marc-specifications.item.get` and the LCCN-setting read permission; link/unlink includes `marc-records-editor.links.suggestion.post`.

A user without the create/edit/derive permission for an action does NOT see that `Actions` menu option at all — absent, not disabled (C422105).

---

## Common Verification Patterns

### Create a new MARC bib record — first-open field inventory (confirmed pattern, C422104)

Verify `001` empty, `005` empty, `008` tag-only initially, `999` empty, LDR groups exactly as documented (read-only 00-04/09-16/20-23 with fixed defaults; dropdowns 05/06*/07*/08/17/18/19 with default values and hover tooltips), `245` with both indicators blank and an empty `$a` (default Create fields = `001, 005, 008, 245, 999` — ui-quick-marc `CREATE_BIB_RECORD_DEFAULT_FIELD_TAGS`). Then verify Close (`x`) and `Cancel` both discard with no record created.

### Non-repeatable field validation (confirmed pattern, C380644, C496193, C496209)

1. Fill required fields (LDR Type/BLvl, red 008 dropdowns, 245 $a) so Save is enabled.
2. Add a second instance of a non-repeatable field (010, 008, 1XX, 245, 005 with data...).
3. Expected: inline `Fail: Field is non-repeatable.` (1XX: `Fail: Field 1XX is non-repeatable.`) under the duplicate, pane stays open.
4. Remove the duplicate, save → success toast; re-open via `View source`/Edit to confirm only the surviving field persisted.

### 008 fixed-field position validation (confirmed pattern, C503013; UIQM-711)

An invalid dropdown selection shows a **separate inline error per invalid position**, all at once under 008; a wrong-length text box (e.g. `Ctry` = 3 chars) gets its own length error. Iterate until zero red positions remain. A text box shorter than its length is padded with `\`, longer is truncated (UIQM-697).

### System-managed field handling (C380699, C496208, C496209, C916264, C594519, C380716; UIQM-781)

- `001` is system-generated (HRID) on save.
- `005` (Trillium+, UIQM-781): the **tag** box of a 005 row is editable like 008's (a mistyped 005 can be re-tagged); the content box is read-only. Multiple 005 **with data** → `Fail: Field is non-repeatable.` on save (C496209). Multiple **empty** 005s, or no 005 at all → record saves and the system generates one 005 (`YYYYMMDDHHMMSS`) (C496209 steps 6-7, C916264, C594519).
- `999`: before Trillium all user-added 999s were stripped on save; from Trillium only `999 ff` is system-managed and other indicator combinations (e.g. `12`) are preserved next to the system `999 ff $s {uuid} $i {uuid}` (C380716). State which release behavior a 999 case targets.

### Delete-fields confirmation flow

Deleting field(s) then saving opens `Delete fields` — verify both `Continue with save` (fields deleted, record updated) and `Restore deleted field(s)` (fields restored, nothing saved).

### Authority-linking gating

Editing a controlled subfield on a linked bib field is blocked (`A subfield(s) cannot be updated because it is controlled by an authority heading.`). Cover Manual and Automated linking per the corpus split (24361/24363/24365 vs 24362/24364/24366).

### LDR position 05 (Record status) drives Instance suppression flags (C1307997; MODQM-509)

Setting **LDR 05 = `d`** and saving sets the Instance `Set for deletion = true`, `Suppressed from discovery = true`, `Staff suppressed = true` (mod-quick-marc create/update docs: `staffSuppress` and `discoverySuppress` set when LDR/05 = d, on Create **and** Edit). Changing back to `c` reverts `Set for deletion` only; the two suppression flags stay true until cleared on the Instance. Assert all three Instance flags.

### Instance fields not mapped from MARC survive a quickMARC save (MODQM-505, MODQM-514)

After editing any MARC field, Instance fields not derived from MARC (statistical codes, administrative notes, nature of content, Instance status term + `status updated`, suppress flags, cataloged date, parent/child instances, previously held) must keep their values; MARC-derived Instance fields absent from the MARC are set to null (update-record.md "SET_TO_NULL"). Good regression check for any Edit case.

### Optimistic locking (section 27186 — C353225, C353226, C367950, C376995)

Two users open the same record; B saves first; A then saves → conflict banner `This record cannot be saved because it is not the most recent version.` with `View latest version` (opens the updated Instance in a new tab), focus on the banner. Since Trillium the check uses the SRS generation, so it also fires when the other user changed a MARC field **not mapped** to the Instance (C367950; UIQM-791, MODQM-478) — not backported to Sunflower (UIQM-791 comment). Also fires when a linked authority is updated while the bib is open (C376995, C397340, C397342).

### Derive — field carry-over rules (C387459, C496210, C499607, C503056, C503160, C494021; ui-quick-marc `FIELDS_TO_CLEAR_FOR_DERIVE`)

- Derive clears `001`, `005`, `010`, `019`, `035` and `999 ff` from the source record. `010` is shown as an empty row (`010 \\ $a`) rather than copied (UIQM-661, C494021).
- `001`/`005`: a manually added duplicate collapses to the single system value on save (C387459, C496210) — Trillium+ 005-with-data duplicates get the non-repeatable Fail instead (UIQM-781; C496210 last updated 2026, re-read before asserting).
- `006`/`007`: repeatable and user-authorable; all rows persist (C499607).
- `245` required: empty/missing `$a` → `Field 245 is required.` at Fail tier (C503056).
- Local control fields `002`/`004`/`009`: save with or without a subfield code (UIQM-670, C503160).

### Manual linking — exact modal & locked-box mechanics (C422126 Create, C365134 Edit)

1. A linkable field shows the `Link to MARC Authority record` icon [en.json `record.link`] as soon as its tag is linkable.
2. The icon opens **Select MARC authority**: `Browse` toggle by default with the heading type pre-selected (Personal name for x00). For 600/610/611, 700/710/711, 800/810/811 that contain `$t` and no `$0`, the default browse option is **Name-title** (UIQM-785). Browse/Search boxes are auto-populated with all controllable subfields of the bib field, in bib order (UIQM-762).
3. Clicking `Link` next to the highlighted result links the field and closes the modal.
4. Locked: tag + indicators, the controlled heading subfields, `$0`; editable: uncontrolled subfields (before and after the controlled box). Row icons become `Unlink from MARC Authority record` and `View MARC authority record` [en.json].
5. Uncontrolled subfields keep the typed order (e.g. `$n ... $t ... $n ...` not grouped — UIQM-720) and controlled subfields follow the authority's subfield order (UIQM-724).
6. **Indicators are preserved after linking** (manual and automated) — regression UIQM-829 reset them to `\`.
7. `$0` format follows the authority's identifier scheme: bare naturalId (`$0 3052044`) or full URI (`$0 http://id.loc.gov/authorities/names/n2008001084`) (C422126, C365134).

### Automated linking — "Link headings" (C388560; links-suggestions.md)

Requires (1) the field's linking rule `autoLinkingEnabled: true` and (2) `$0` exactly matching an authority `naturalId`. One click processes all eligible fields; matched fields link, unmatched fields get the "must be set manually" toast (both toasts can appear). Pre-existing non-controllable subfields (`$j`, `$2 fast`) are preserved; controllable heading subfields (`$a`, `$d`) are replaced by the authority heading. The button stays enabled for further passes.

### Version history (MARC Bib) (C663247, C692071, C692230, C692168; UIQM-748/761)

- Clock icon (tooltip `Version history`) next to `Actions` on `View source`; opens a pane with an exact `N version(s)` counter; `Actions` is disabled while it is open.
- Cards newest → oldest: top = "Current version" (bold+italic) + `Changed` link; middle = `Changed` only; bottom = "Original Version". `Load more` only when more cards exist than the configured count.
- `Changed` opens a modal (`[Date], [Time]`, `Source`) with table `Action | Field | Changed from | Changed to` (sorted by Action); blank indicators render as whitespace; close via `X` or `Close`.

### Consortia — shared MARC Bib edit (C405507)

Member-tenant user can edit a Shared bib directly; toast is the standard edit toast (`...is in process. Changes may not appear immediately.` — same string as non-ECS edit, see Save-flow callouts); `Record last updated` source = editing user, visible from every tenant.

### Consortia — validation rules, linking rules and Derive (UIQM-699, UIQM-766, UIQM-768, UIQM-773; C552462, C514879)

- Shared record (create on Central or edit from Central/Member) → validated against **Central** rules; Local record → against the **Member** tenant's rules. Central and Member rules can contradict (C552462 `Field 700 is required.`).
- Shared bib edited from Central or Member uses **Central** linking rules (UIQM-773, Trillium).
- Deriving from a Shared record on a Member tenant produces a **Local** record (pane `Derive a new local MARC bib record`), validated with **Member** rules; `Save & keep editing` reopens it on the Member tenant (UIQM-766, UIQM-768, C514879).

### Duplicate LCCN check (C514849, C514879, C569611, C589505-C589509; UIQM-658/709/786; validators.js)

- Runs only when the tenant setting is on and `010 $a` has a value (no 010 → saves normally, C523560/C514844).
- Matches on other records' **010 $a only** — a value equal to another record's Canceled LCCN (`010 $z`) does **not** raise Fail or Warn; the record saves (UIQM-786; C514849 last step, C514879 steps 3 and 7).
- Prefix/spacing variants of the same LCCN (`vp  58020560`, `vp58020560`, `vp 58020561`) all match (C514849).
- Records with **both** Staff suppressed and Suppressed from discovery (i.e. set for deletion) are excluded; a record with only one of the flags still counts as a duplicate (UIQM-709; C569611, C589505-C589509).
- Edit excludes the record itself; Create/Derive do not filter by id (validators.js).
- In ECS a Local derive is blocked by a match on both Shared and Local records (C514879).

---

## ECS / Multi-Tenant Notes

Sections 27635 (Create), 26856 (Derive), 27434 (Edit), 24361>28950 / 24362>28952 / 24363>27636 / 24364>27638 / 24365>27639 / 24366>28951 (linking), 61630 (Version history). Use Central + Member-1 (+ Member-2 for cross-tenant visibility); state the active affiliation per step. Key behaviours: Shared vs Local validation rules and linking rules (see above); Derive from Shared on Member → Local; Version history resets to one "Shared" card when a Local record is shared (C692169); cross-tenant `Source` on a card is a hyperlink only if the viewer can view users (C692168, C1434628); a MARC permission granted through a permission set in a Member tenant must be enough to edit a shared bib (UIQM-735).

---

## Existing automated coverage

- **Karate** — `folio-integration-tests/mod-quick-marc` (runner `src/test/java/org/folio/QuickMarcApiTest.java`): `features/quick-marc-bib-records.feature` (GET by instanceId, PUT linked/unlinked fields, double update, create bib, 404/invalid id, illegal fixed-field length, id mismatch, "Record specification violation", 035 normalise / 003 removal on create, 035/003 unchanged on update + optimistic locking), `quick-marc-edit-bib-record.feature`, `marc-bib-update.feature`, `quick-marc-linking-records.feature` (authority 100/010 update propagates to bib, authority delete removes `$9`), `quick-marc-handle-literal-dollar.feature`, `quick-marc-tags-order.feature`, `quick-marc-update-linked-authority-records.feature`. No dedicated scenario for `POST /records-editor/validate` or `/links/suggestion` found in the feature titles.
- **Cypress** — `stripes-testing/cypress/e2e/marc/marc-bibliographic/`: `create-new-marc-bib/` (~87 specs), `derive-marc-bib/` (~104), `edit-marc-bib/` (~198), `validation-rules/` (~101, incl. `specification-storage-fields-api/`), `optimistic-locking/` (7); consortia in `cypress/e2e/consortia/marc/marc-bibliographic/` (~98). Fragment: `cypress/support/fragments/quickMarcEditor.js` (callout constants, optimistic-locking banner, Link headings popover).

---

## Edge cases from bugs (regression candidates)

- 008 positions 00-05 blank on import must not shift the remaining positions left (MODQM-515); `Entered` (00-05) is auto-filled with today's date on save if blank/non-numeric/`000000` (update-record.md).
- 008 text boxes must not render as empty dropdowns (MODQM-519).
- Empty `ELvl` (LDR 17) is auto-filled with `u` on save, one message not three (UIQM-685).
- 007 type `g` position 06 (`MfS`) accepts one character only (UIQM-712); 007 type `h` RRR/RR box accepts 4 characters (UIQM-708).
- A field ending with an empty subfield code (e.g. trailing `$b`) saves; empty subfields are dropped (MODQM-445).
- Upper-case subfield code (`$A`) is converted to lower case on save (MODQM-477, mod-quick-marc 8.0.0 NEWS).
- Literal `$` is entered as `{dollar}` (docs QuickMARC.md; Karate `quick-marc-handle-literal-dollar.feature`).
- Polish special characters (e.g. `ś`) must not close the editor (UIQM-592).
- Validation of uncontrolled subfields added after linking still runs (UIQM-704).
- New 008 Continuing-resources `SrTp` (008/21) options `a`, `i`, `q` with labels `Activity report`, `Serial zine`, `Serial podcast` [en.json `record.fixedField.*`]; records imported with these values must no longer fail (UIQM-836/837/838, MODQM-528/530/531; C451570, C451571 updated); backported to Trillium 11.0.2 and Sunflower 10.0.5.
- 008/25 value `r` allowed (MODQM-511, UIQM-818).
- Validation rules apply only to quickMARC create/edit/derive, not to Data Import (docs QuickMARC.md).

---

## Key Business Rules for Test Cases

> One rule per line; source in parentheses.

1. Fail vs Warn are distinct: Warn lets the user re-save as-is; Fail hard-blocks until corrected — a negative case must name the tier. (C496208, C496193; mod-record-specifications validator severities; UIQM-631)
2. Required field, non-repeatable field/subfield, invalid tag/indicator format, invalid LCCN format and 1XX rules are Fail; undefined field, undefined indicator code and undefined subfield code are Warn. (mod-record-specifications `*RuleValidator.severity()`; Confluence 1407631)
3. Only LDR positions 5, 6, 7, 8, 17, 18, 19 are editable for bib; Holdings/Authority have their own lists and messages. (ui-quick-marc `LEADER_EDITABLE_BYTES`; en.json `record.error.leader.forbiddenBytes.*`)
4. `001`, `005` and `999 ff` are system-managed: 001 is the generated HRID; a 005 tag box is editable (Trillium+) but its content is system-generated, with multiple 005s holding data failing as non-repeatable and empty/missing 005 regenerated; `999 ff` is regenerated while other 999 indicator pairs are kept (Trillium+). (C380699, C496209, C916264, C380716; UIQM-781)
5. Non-repeatable fields (010, 008, 1XX, 245, 005 with data) raise an inline non-repeatable error, never a silent overwrite or silent duplicate. (C380644, C496209; rspec `validation.nonRepeatableField`)
6. Subfields controlled by a linked authority are locked; typing a controlled subfield code into an uncontrolled box raises the "controlled by an authority heading" Fail. (en.json `record.error.fieldIsControlled`; UIQM-694; C422126)
7. Missing the quickMARC create/edit/derive capability hides the matching `Actions` option entirely. (C422105)
8. Optimistic locking compares the SRS generation (`sourceVersion`): a stale save gets HTTP 409 and the conflict banner `This record cannot be saved because it is not the most recent version.` + `View latest version`, also when only non-Instance-mapped MARC fields changed (Trillium+). (C353225, C367950; MODQM-478, UIQM-791; update-record.md)
9. `This record has been deleted by another user...` is shown only for a `NOT_FOUND` save error (record deleted), not for a concurrent edit. (QuickMarcEditor.js; en.json `record.save.error.notFound`; UIQM-825)
10. LDR 05 = `d` on Create or Edit sets Instance `Set for deletion`, `Suppressed from discovery` and `Staff suppressed` to true; reverting to `c` clears only `Set for deletion`. (C1307997; MODQM-509; create-record.md, update-record.md)
11. Every rule here is Bib-specific; Holdings and Authority rules live in `marc-holdings.md` and `marc-authority.md`. (section structure 16714/17151)
12. Derive clears 001, 005, 010, 019, 035 and 999 ff from the source; 006/007 rows are repeatable and all persist. (ui-quick-marc `FIELDS_TO_CLEAR_FOR_DERIVE`; UIQM-661; C494021, C499607)
13. `245` is required on Create, Derive and Edit: missing/empty `$a` → `Field 245 is required.` at Fail tier. (C503056; rspec `validation.missingField`)
14. Manual linking locks tag, indicators, controlled heading subfields and `$0`, keeps uncontrolled subfields editable, and replaces the link icon with Unlink / View MARC authority record icons. (C422126, C365134; docs QuickMARC.md)
15. `Link headings` links only fields whose `$0` matches an authority `naturalId` and whose linking rule has auto-linking enabled; unmatched fields are reported in the "must be set manually" toast; non-controllable subfields survive, controllable ones are replaced. (C388560; links-suggestions.md)
16. Version history: clock icon → pane with `N version(s)`; top card "Current version" + `Changed`, middle cards `Changed`, bottom "Original Version"; `Changed` opens the `Action | Field | Changed from | Changed to` modal; `Actions` is disabled while the pane is open. (C663247, C692071, C692230; UIQM-748, UIQM-761)
17. A Member-tenant user can edit a Shared bib directly; the change and `Record last updated` source show from every tenant. (C405507)
18. Shared records validate against Central rules and Local records against the Member tenant's rules; Central and Member rule sets may contradict. (UIQM-699; C552462)
19. Sharing a Local bib resets its Version history to a single "Shared" card; cross-tenant `Source` is a hyperlink only for users who can view users. (C692169, C692168, C1434628)
20. The duplicate-LCCN check (tenant setting on) blocks save with `Fail: 010 $a already exists.` when another record's `010 $a` matches; a match on another record's Canceled LCCN (`010 $z`) saves with no Fail or Warn. (C514849, C514879; UIQM-786)
21. Duplicate-LCCN check ignores records that are both staff suppressed and suppressed from discovery; a record with only one of these flags still counts. (UIQM-709; C569611, C589505, C589509; validators.js)
22. Duplicate-LCCN check runs only if `010 $a` has a value; a record without 010 saves normally with the check on. (C523560, C514844; UIQM-658)
23. The duplicate-LCCN setting has no UI; it is the mod-settings entry `lccn-duplicate-check` (`duplicateLccnCheckingEnabled`) under scope `ui-quick-marc.lccn-duplicate-check.manage`. (UIQM-678; C514849 preconditions)
24. Create shows `...is in process...` then `Record created.`; Derive shows `Creating record may take several seconds.` then `Record created.`; Edit shows `...is in process. Changes may not appear immediately.` (useSumbitRecord.js; Cypress quickMarcEditor.js)
25. On Create, LDR `Type` and `BLvl` carry a required asterisk and 008 boxes appear only after both are selected. (UIQM-666; docs QuickMARC.md; C514849)
26. All invalid 008 dropdown positions are reported together, one inline Fail per position. (UIQM-711; C503013)
27. Empty LDR `ELvl` is auto-filled with `u` on save. (UIQM-685)
28. Local control fields 002/004/009 save without a subfield code and are not auto-given `$a`. (UIQM-670; C503160)
29. Saving through quickMARC keeps Instance fields not mapped from MARC (statistical codes, administrative notes, nature of content, status term/date, suppress flags). (MODQM-505, MODQM-514; update-record.md)
30. Upper-case subfield codes are lower-cased and empty subfields are dropped on save. (MODQM-477, MODQM-445; mod-quick-marc NEWS 8.0.0)
31. On save of a bib record `003` is removed and `035` is normalised; Create adds `001` (HRID) and `999 ff $i`. (create-record.md, update-record.md; Karate quick-marc-bib-records.feature)
32. Manual and automated linking keep the field's indicator values. (UIQM-829; C380742)
33. For 600/610/611, 700/710/711, 800/810/811 with `$t` and no `$0`, the authority plugin opens with Name-title browse; other linkable fields with `$t` (e.g. 630/650/651/655/730/830) are not affected. (UIQM-785 + QA comment)
34. The authority plugin's Browse/Search box is pre-filled with all controllable subfields of the bib field, in bib order. (UIQM-762)
35. `Link headings` shows an info-tip with the fixed text and a `Learn more` link; the button is hidden when auto-linking is off for all heading types. (UIQM-807; C422143, C389484)
36. Deriving from a Shared bib on a Member tenant creates a Local record validated with the Member tenant's rules. (UIQM-766, UIQM-768; C514879)
37. Editing a Shared bib from Central or Member uses Central linking rules. (UIQM-773)
38. If validation takes over 2 seconds a `MARC validation rules check` modal is shown. (UIQM-632; en.json `validation.modal.*`)
39. `Cancel`/`X`/`Esc` with unsaved changes shows the unsaved-changes modal even while Warn/Fail errors are displayed. (UIQM-743)
40. MARC validation rules apply only to quickMARC create/edit/derive, not to Data Import. (docs QuickMARC.md)
41. 008 continuing-resources `SrTp` accepts `a`, `i`, `q` (`Activity report`, `Serial zine`, `Serial podcast`); such records must save without a Fail. (MODQM-530, UIQM-838; en.json; C451570, C451571)

---

## Authoring style (measured 2026-07-23)

MARC Bib / quickMARC: **`Func` ~79%** / Other ~21%, median ~8 steps, `User Journey` ~2% (~21% of cases are ≥15 steps). Use `Functional`. Preconditions carry the source MARC bib (and any linkable authority) and, for LCCN cases, the mod-settings setup steps. Longer cases walk edit→save→verify-Instance/linked-authority; field-validation and element checks stay compact.

---

## Known Gaps / Items to Verify

- [ ] **Stale Cypress constants**: `quickMarcEditor.js` still holds legacy strings (`Record cannot be saved with more than one field 245.`, `Record cannot be saved. Can only have one MARC 001.`, `Record cannot be saved with more than one 010 field`, `Record cannot be saved without 008 field`) whose bib keys are no longer referenced in ui-quick-marc `src` — confirm in env which wording a bib user sees before reusing them; backend `Field is non-repeatable.` / `Field {tag} is required.` are case-confirmed.
- [ ] **005 rule vs older cases**: UIQM-781 (Trillium) changed 005 handling, but C496208/C496210 (Create/Derive multiple 005) were re-read 2026-11-25: C496208 (Create) and C496210 (Derive) say that typing `005` in the new row's tag box makes the row "read only" and that after save only one system-generated 005 remains; C496209 (Edit) still says the row becomes "read only" after typing 005 while UIQM-781 makes the tag editable — verify exact behaviour (tag vs content box) in env.
- [ ] **docs.folio.org lag**: QuickMARC.md (2026-05-28) still says "Fields 005 and 999ff are system supplied and cannot be edited or deleted" — conflicts with UIQM-781 for the 005 tag box.
- [ ] **validate endpoint severity**: mod-quick-marc `docs/features/validate-record.md` says only `ERROR` issues are returned, but the UI shows Warn issues (UIQM-631/691, C496208) and mod-record-specifications emits WARN severities — confirm against a `/records-editor/validate` response before writing an API case.
- [ ] **LCCN structure rule** (`010 $a is in an invalid format.`, MODQM-447) is LoC-oriented; whether it is enabled by default in the bib specification on every tenant is not confirmed — check `GET /specification-storage/specifications/{id}/rules` first.
- [ ] **Delete-fields modal / Edit toast after 1XX change**: `record.save.updatingLinkedBibRecords` is fired when 1XX/010 of an **authority** changes; not expected on bib edits — not case-confirmed for bib.
- [ ] **TestRail sweep was partial**: read C353225, C496208, C496209, C496210, C514849, C514879 in full plus titles/refs for sections 27186 and for refs UIQM-786/807/785/829/838/791/661/709, MODQM-509 and the 005 title search; cases created after 2026-07-21 were listed only for the first 250 suite-wide results (MARC bib hits: none new beyond C1563888/C1434628 from the previous delta).
- [ ] UIQM-834 (ECS 852 $b after ownership update) from the 2026-11-07 delta is a **MARC Holdings** issue — tracked in `marc-holdings.md`, not here.
- [ ] Karate has no scenario titled for `/records-editor/validate` or `/records-editor/links/suggestion` — candidates for Karate-first coverage.

> Resolved in this refresh: Eureka capability names for Derive and Link/unlink (docs QuickMARC.md, C514879); Consortia Derive sub-flow (C514879, UIQM-766/768); Edit-flow linking (C365134 plus UIQM-829/785/762); optimistic-locking message (corrected to the conflict banner, C353225); non-shared edit toast (same `...is in process...` string — useSumbitRecord.js); duplicate-LCCN Canceled-LCCN behaviour (corrected: no Warn, saves — UIQM-786, C514849).
