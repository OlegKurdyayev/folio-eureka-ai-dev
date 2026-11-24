---
area: linked-data
generated_on: 2026-11-24
generator_version: manual (build-app-context method, full refresh)
jira_status: ok
---

# FOLIO Linked Data Editor (Marigold) — Business Logic Context

> Built: 2026-11-24 · Release: Umbrellaleaf R2 2026 (master: ui-linked-data 2.0.4 and ui-ld-folio-wrapper 2.0.6 per package.json; mod-linked-data NEWS "v3.0.0 (IN PROGRESS)"; previous release Trillium R1 2026 = mod-linked-data 2.0.x / UILD 2.0.x, Sunflower = 1.x) · Builder: build-app-context method, manual
> Previous build: 2026-11-07.
> Sources: TestRail project 14 suite 21: section 26752 "Marigold" (user journeys C624167…C721761) and sub-sections 118240 (LD Authority, UILD-825/826 cases), 66260 (RDF Import), 29721 (locale); C1292045 (Marigold administrator role, section 40728).
> GitHub (master, 2026-11-24): folio-org/ui-linked-data (`translations/ui-linked-data/en.json`, CHANGELOG), folio-org/ui-ld-folio-wrapper (`package.json` stripes.permissionSets / okapiInterfaces), folio-org/mod-linked-data (`descriptors/ModuleDescriptor-template.json`, `ValidationMessages.properties`, `errors.yml`, validators in `src/main/java/org/folio/linked/data/validation/`, `src/main/resources/profiles/*.json`, NEWS, README), folio-org/mod-linked-data-import (ModuleDescriptor), folio-org/ui-inventory (`translations/ui-inventory/en.json`), folio-org/folio-integration-tests (`mod-linked-data/`), folio-org/stripes-testing (`cypress/e2e/linked-data/`, `cypress/support/fragments/linked-data/`, `cypress/support/dictionary/marigoldCapabilities.js`).
> Jira (anonymous REST): UILD + MODLD Done/Closed since 2025-01 (851 issues; ~60 rule-bearing stories/bugs read in detail). Note: most UILD/MODLD issues carry no fixVersion, so scope was taken by resolution date.
> Confluence: "Trillium R1 2026 New apps" (mod-linked-data-import), "Role Names and Keycloak Realm Migration".
> Case titles start with the Jira keys in brackets, e.g. "[UILD-826] PERSON - Authority profile, create/edit page" — follow that convention in this area.

## What is the Linked Data Editor

**Marigold** (app title from `ui-ld-folio-wrapper` en.json `meta.title`; UI module ui-linked-data, wrapped by ui-ld-folio-wrapper, route `/linked-data-editor`, home `/linked-data-editor/search`; backend mod-linked-data, bulk import mod-linked-data-import) is a BIBFRAME editor over a linked-data **graph**. Editable resource types: **Work**, **Instance**, **Hub**, **Authority** (Person, Family, Organization, Meeting, Jurisdiction, Place, Temporal, Topic, Form; Complex Subject planned — MODLD-1098). Instances created in Marigold become Inventory instances with source **LINKED_DATA** (`ResourceSource` enum in mod-linked-data: `LINKED_DATA`, `MARC`); MARC is derived from the graph ("View MARC"), and MARC bib records can be converted into the graph from Inventory.

## Architecture / Integrations

- mod-linked-data → Kafka → **mod-search** (Work index `linked-data.work`, Hub index `linked-data.hub`, Authority index `linked-data-authority` — MODLD-1082/1099); mod-search consumes and indexes asynchronously → hence the "Expect a short delay" toasts (mod-linked-data README "Search module").
- mod-linked-data → Kafka (instance ingress) → **mod-inventory** creates/updates the Inventory instance; mod-inventory instance events and SRS source-record domain events flow back (README "Inventory module", "Source Record Storage module").
- **mod-settings**: LCCN duplicate-check setting and base URL (`SettingsClient` → `settings/entries`, `base-url`); **mod-record-specifications**: LCCN pattern rule (`specification-storage` interface); cached (`CACHE_TTL_SETTINGS_ENTRIES_MS`, default 18000000 ms) — Karate tests clear caches via `DELETE /linked-data/admin/caches` before re-checking a changed setting (`lccn-deduplication.feature`, `lccn-pattern-validation.feature`).
- **mod-linked-data-import**: S3-based bulk RDF import (needs S3 env vars — Confluence "Trillium R1 2026 New apps"); results come back to mod-linked-data over Kafka.
- Lightweight Work/Instance (`LightResource`) created from MARC 775/776/490 are NOT sent to Inventory or mod-search and are excluded from reindex (MODLD-980, MODLD-1071).

## Profiles (resource type → profile id)

| Id | Profile name | Resource type | Source |
|---|---|---|---|
| 2 | Books | Work (`library/Books`) | `src/main/resources/profiles/work-books.json` |
| 6 | Serials Work | Work (`library/ContinuingResources`) | `work-serials.json` |
| 3 | Monograph | Instance | `instance-monograph.json` |
| 4 | Rare Books | Instance | `instance-rare-books.json` |
| 5 | Serials | Instance | `instance-serials.json` |
| 7 | Hubs | Hub | `hub.json` |
| 8–16 | Family, Form, Jurisdiction, Meeting, Organization, Person, Place, Temporal, Topic | Authority | `authority-*.json`, MODLD-1083 |

Create URL pattern for an authority profile: `/linked-data-editor/resources/create?type=authority&profileId=13` (UILD-838).

## UI structure

- Search page tabs: Resources (Works/Instances) / Hubs / Authorities. Resources: basic search, Advanced search (AND/OR/NOT, Starts with / Contains all / Exact phrase), results with Work preview, "Actions" for Work/Instance. Search state is preserved when returning from an edit page (UILD-702, C466101, C468225).
- Authorities tab (C1395177): Keyword selected by default; Source list "Local Marigold Authority" (default), "Local MARC Authority", "Local LoC Children's Subject heading"; Actions = "New authority", "Manage profile settings"; result columns Label, Type, Identifiers, Authorized/Reference, Source, Action ("Edit" for Local Marigold; "Import/Edit" or "Edit" for other sources). Search options: Keyword, Identifier (All), Family, Form, Jurisdiction, Meeting, Organization, Person, Place, Temporal, Topic, Complex Subject, Uniform Title, Name-title.
- Hubs tab: sources "Library of Congress" / "Local" (`ld.source.*`); LoC result → "Import/Edit" → hub preview (Creator, Title, Language) with Cancel / Continue → Continue imports the hub and opens Edit hub (UILD-725/726, MODLD-1053).
- Edit pages for Work / Instance / Hub / Authority: "Save & close", "Save & keep editing", "Cancel", Duplicate (titles get "(DUPLICATE WORK)" etc.), "View MARC", "Export instance RDF file", "Inventory View" (Instance Actions → Inventory instance preview in the same tab, UILD-768), unsaved changes modal. The Work "Actions" button sits inside the Work edit area, separate from Instance Actions (UILD-765).
- Authority assignment: "Assign authority" / "Change", modal with Search / Browse and, for Subject of the Work, Authorities / Hubs toggle (UILD-711); hubs are imported on "Import/Assign" click (UILD-732); "Change authority" opens the modal in the view matching what is assigned (UILD-730).
- Profiles: "Choose a resource profile" modal with "Set as my default"; "Change work/instance profile" from Actions (UILD-577); "Manage profile settings" with multiple named settings per profile, drag & drop between "Selected components" / "Unused components".
- Import: "Import resources" / "Import hubs" via JSON file or URL; import log downloaded.
- Inventory integration: "Edit resource in Marigold" in Inventory instance Actions (ui-inventory en.json `editInMarigold`), permission-gated; MARC-sourced instances open a read-only preview with "Continue" which converts the record into the graph.

## API surface (mod-linked-data `linked-data 2.0`, `linked-data-profile 1.0`, `linked-data-vocabularies 1.0`, `linked-data-batch-job 1.0`, `linked-data-admin 1.0`)

> From `descriptors/ModuleDescriptor-template.json` (master).

| Method + path | permissionsRequired |
|---|---|
| POST `/linked-data/resource` | `linked-data.resources.bib.post` |
| GET / PUT / DELETE `/linked-data/resource/{id}` | `linked-data.resources.bib.get` / `.put` / `.delete` |
| GET `/linked-data/resource/{id}/marc` (View MARC) | `linked-data.resources.bib.marc.get` |
| GET `/linked-data/resource/{id}/rdf` (Export instance RDF) | `linked-data.resources.rdf.get` |
| GET `/linked-data/resource/{id}/graph` | `linked-data.resources.graph.get` |
| GET `/linked-data/resource/metadata/{inventoryId}/id` | `linked-data.resources.bib.id.get` |
| GET `/linked-data/inventory-instance/{inventoryId}/import-supported` | `linked-data.resources.support-check.get` |
| GET `/linked-data/inventory-instance/{inventoryId}/preview` | `linked-data.resources.preview.get` |
| POST `/linked-data/inventory-instance/{inventoryId}/import` | `linked-data.resources.import.post` |
| POST `/linked-data/authority-assignment-check` | `linked-data.authority-assignment-check.post` |
| POST `/linked-data/import/file`, POST `/linked-data/import/url` | `linked-data.import.file.post`, `linked-data.import.url.post` |
| GET / POST `/linked-data/hub?hubUri=…` (preview / import LoC hub) | `linked-data.hub.preview.get` / `linked-data.hub.import.post` |
| GET `/linked-data/vocabularies/{vocabularyName}` | `linked-data.vocabularies.item.get` |
| GET `/linked-data/profile/{id}`, GET `/linked-data/profile/metadata` | `linked-data.profiles.item.get`, `linked-data.profiles.metadata.get` |
| GET / POST / DELETE `/linked-data/profile/preferred` | `linked-data.profiles.preferred.get` / `.post` / `.delete` |
| GET / POST `/linked-data/profile/{profileId}/settings` | `linked-data.profiles.settings.list.get` / `.post` |
| GET / PUT / DELETE `/linked-data/profile/{profileId}/settings/{id}` | `linked-data.profiles.settings.get` / `.put` / `.delete` |
| GET / POST / DELETE `/linked-data/profile/{profileId}/preferred` | `linked-data.profiles.settings.preferred.get` / `.post` / `.delete` |
| POST `/linked-data/reindex/full`, `/linked-data/reindex/incremental` (`?resourceType=WORK\|HUB`) | `linked-data.batch.reindex.full.post`, `linked-data.batch.reindex.incremental.post` |
| POST `/linked-data/graphCleaning` (also a `_timer`) | `linked-data.batch.graph.cleaning.post` |
| GET `/linked-data/batch/status?jobExecutionId=` | `linked-data.batch.status.get` |
| DELETE `/linked-data/admin/caches` | `linked-data.admin.caches.delete` |

mod-linked-data-import (`linked-data-import 1.0`): POST `/linked-data-import/files` (`linked-data-import.files.collection.post`), POST `/linked-data-import/start?fileName=…&defaultWorkType=…` (`linked-data-import.start.post`), GET `/linked-data-import/jobs/{jobExecutionId}` (`linked-data-import.jobs.item.get`), GET `…/jobs/{id}/failed-lines` (`linked-data-import.jobs.failed-lines.collection.get`), PUT `…/jobs/{id}/stop` (`linked-data-import.jobs.item.stop.put`).

Error codes (`src/main/resources/errors.yml`): `not_found` 404, `mapping` 400, `not_supported` 400, `required` 400, `already_exists` 400, validation 400 with a code from `ValidationMessages.properties`, `generic_bad_request` 400, `server` 500, `failed_dependency` 424. Validation codes: `required_primary_main_title`, `required_authority_name`, `lccn_does_not_match_pattern`, `lccn_not_unique`, `required_series_title`, `wrong_resource_type`, `instance_no_single_work`, `profile_settings_name_not_unique`. The UI maps codes to `ld.<code>` translation keys (e.g. `ld.lccn_not_unique`, `ld.failed_dependency`, `ld.not_found`).

## Status Values

- Identifier "Invalid/Canceled?" (Instance and Authority identifiers): canceled or invalid (cancinv), current (current), incorrect (incorrect) — local vocabulary replacing LoC mstatus (MODLD-1041, C1385658).
- Authority identifier type: LCCN (default) / Other (MODLD-1083, C1385658).
- Inventory instance source: LINKED_DATA / MARC (`ResourceSource.java`).
- Import job status (mod-linked-data-import): COMPLETED, STOPPED (C1292045, C1307933); counters `linesRead`, `linesMapped`, `linesCreated`, `linesUpdated`, `linesFailedMapping`, `linesFailedSaving`, `savingComplete`.
- Batch job (`/linked-data/batch/status`): `jobName` reindexJob / graphCleaningJob, `reindexType` FULL / INCREMENTAL, `status` COMPLETED (C1292045, README).
- Authority assignment targets (Karate `authority-assignment-check.feature`): CREATOR_OF_WORK, SUBJECT_OF_WORK, DEGREE_GRANTING_INSTITUTION.

## Settings that change behaviour

| Setting | Where | Effect | Source |
|---|---|---|---|
| LCCN duplicate check | mod-settings scope `ui-quick-marc.lccn-duplicate-check.manage`, key `lccn-duplicate-check`, property `duplicateLccnCheckingEnabled` (same setting as quickMARC) | Enables `lccn_not_unique` check on Instance save | `LccnUniquenessValidator.java`, `lccn-deduplication.feature` |
| LCCN pattern validation | MARC bib validation rule with code `invalidLccnSubfieldValue` (mod-record-specifications) enabled | Enables `lccn_does_not_match_pattern` | `LccnPatternValidator.java`, `lccn-pattern-validation.feature` |
| Authority File ("Settings > Authority File" as named in MODLD-922; exact menu path to verify) | base URL for assigned-authority URIs (024 $0/$2) | Authority source names in lookup results come from the Authority File | MODLD-922, UILD-764 |
| User preferred profile / profile settings | per user, per profile (`/linked-data/profile/...`) | Default profile in create modal; component order/visibility in editor | MODLD-745, MODLD-1040, UILD-684 |

## Capability Sets (Eureka) and roles

Roles used by the team (stripes-testing `marigoldCapabilities.js`, C1292045, MODLD-1053, MODLD-1071): **"Cataloger"** + **"Cataloger - Marigold"** for catalogers; add **"Marigold Administrator"** for batch/admin APIs.

| Action | Capability / set | Source |
|---|---|---|
| Open Marigold, catalog Works/Instances/Hubs, profiles, import file/URL | `Data - UI-Marigold Cataloger - Manage` (set; legacy `ui-marigold.cataloger.all` "Marigold: Cataloger permissions", replaces `ui-linked-data.all`) | ui-ld-folio-wrapper package.json, marigoldCapabilities.js |
| App visible | `Settings - Module Ld-Folio-Wrapper Enabled - View` (`module.ld-folio-wrapper.enabled` "Linked Data: Module is enabled") | same |
| Create/assign MARC authorities from Marigold flows | `Data - UI-Marc-Authorities Authority-Record - View/Create/Edit`, `Data - UI-Quick-Marc Quick-Marc-Authorities-Editor - Manage` (part of "Cataloger - Marigold") | marigoldCapabilities.js |
| Inventory side ("Edit resource in Marigold" needs Marigold capabilities) | Cataloger role incl. `Data - UI-Inventory Instance - Edit`, `Data - UI-Quick-Marc Quick-Marc-Editor - Manage` | marigoldCapabilities.js, C651425 |
| Reindex, graph cleaning, cache clear, batch status | `Procedural - Linked-Data Batch Reindex Full/Incremental - Execute`, `Procedural - Linked-Data Batch Graph Cleaning - Execute`, `Data - Linked-Data Admin Caches - Delete`, `Data - Linked-Data Batch Status - View` | marigoldCapabilities.js (Marigold Administrator), C1292045 |
| Bulk RDF import | `Data - Linked-Data-Import Files Collection - Create`, `Procedural - Linked-Data-Import Start - Execute`, `Data - Linked-Data-Import Jobs Item - View`, `Data - Linked-Data-Import Jobs Failed-Lines Collection - View`, `Data - Linked-Data-Import Jobs Item Stop - Edit` | same |

## ECS / Multi-Tenant Notes

- Marigold instances can be created in the central tenant and enriched with holdings in a member tenant (Cypress `create-instance-ecs.cy.js`, C736677 — tagged `draft`).
- OAI-PMH harvests shared LINKED_DATA instances with member holdings/items (TestRail C1347155, C1347156; Karate `edge-oai-pmh/.../consortia/features/linked-data-*.feature`, C688749–C688764).
- No other ECS-specific Marigold cases found.

## Existing automated coverage

- Karate (`folio-integration-tests/mod-linked-data/src/main/resources/promin/mod-linked-data/features/`, runner `linked-data-junit.feature`, `ModLinkedDataCriticalPathExtendedSmokeTest`): authority-assignment-check, authority (create/update), create-bib-api (resource, inventory-outbound, search-outbound, MARC-derived record, RDF, graph), graph-cleaning, hub (connect work to hub, hubs as subject, preview/import hub), import-bib (MARC → graph), import-rdf, import-rdf-bulk, lccn-pattern-validation, validation/lccn/deduplication, profiles (by id, metadata by resource type, preferred profile, profile settings), reindex, suppress-flags, update-instance-api, vocabulary. Also `mod-data-export/.../export-for-linked-data.feature`, edge-oai-pmh consortia LINKED_DATA features.
- Cypress (`stripes-testing/cypress/e2e/linked-data/`, fragments in `cypress/support/fragments/linked-data/`): user journeys C624167 create work, C624234/C552501 duplicate work, C624280/C552492 duplicate instance, C627245 edit + MARC derived record, C633470 new MARC authority, C651425 no LDE permissions, C651426 non-monograph format, C656342 cancel without saving, C692195 comparison mode, C721761 duplicate for work with multiple instances, C736677 ECS; "LD Edit" field/MARC checks (C468168, C490913, C499692, C423954, C436879, C466293, C477531, C466260, C1307932, C829885, C1282818, C446070), C466101/C468225 return to search, C496124 unsaved changes modal, C523770 preview + Actions, C1259767 profile settings multiple users, C491276 header icon; hubs C1030061 (LoC import/edit), C987719/C1030042 (local create/edit). Bulk edit: C651537 LINKED_DATA instances rejected.

---

## Key Business Rules for Test Cases

> One rule per line, source in parentheses.

### Save, toasts, async visibility
1. Successful create shows `ld.rdSaveSuccess`, update shows `ld.rdUpdateSuccess`; both warn of a delay — verify Inventory/search only after indexing (en.json; UILD-703).
2. A Marigold-created instance must appear in Inventory with source LINKED_DATA and be searchable; a MARC instance converted to LD must switch its source from MARC to LINKED_DATA (UILD-858, MODINV-1414).
3. Toasts are not auto-dismissed; they stack (newest on top), expand on hover/focus, and are cleared on view change (UILD-755).
4. On re-opening a saved record, only "Cancel" is enabled; "Save & close" / "Save & keep editing" enable only after a change (UILD-586; C1385658 for a new Authority page).
5. Leaving an edited form (Cancel, X, switching Work↔Instance, page reload) prompts the unsaved changes modal ("Unsaved changes" / "Would you like to save your changes before proceeding?"); "Continue without saving" discards (UILD-226, UILD-370, UILD-539, C496124, C656342).
6. Closing/cancelling/saving an edit page returns the user to the previous search result set (UILD-702, C466101, C468225).
7. Date of creation and Local resource ID in Admin metadata are filled only after processing; the UI shows a placeholder until then (UILD-641; placeholder text `ld.placeholder.processing`).

### Validation (backend codes → UI messages)
8. Work/Instance cannot be saved without a main title → `required_primary_main_title` / `ld.required_primary_main_title` (ValidationMessages.properties; en.json).
9. Each series needs a title → `required_series_title` / `ld.required_series_title` (MODLD-786; en.json).
10. Authority cannot be saved without Name (blank-only values also rejected) → 400 `required_authority_name` (MODLD-1107, `AuthorityNameDtoValidator.java`, C1385658).
11. An Instance must reference exactly one Work → `instance_no_single_work`; importing an instance already tied to another Work fails (`linesFailedSaving: 1`) (MODLD-987, `InstanceSingleWorkDtoValidator.java`, C1307933).
12. LCCN pattern check runs only when the MARC validation rule `invalidLccnSubfieldValue` is enabled AND the LCCN status is current or empty; value is normalized first, then must match Structure A (`( {3}|[a-z][ a-z]{2})\d{8} `) or Structure B (`( {2}|[a-z][ a-z])\d{10}`) → `lccn_does_not_match_pattern` (`LccnPatternValidator.java`, `lccn-pattern-validation.feature`).
13. LCCN duplicate check runs only when the quickMARC LCCN duplicate setting is on, only for Instances with a current/empty-status LCCN; suppressed instances and the instance being edited are excluded → `lccn_not_unique` (`LccnUniquenessValidator.java`, `lccn-deduplication.feature`, UILD-563).
14. If mod-search is unreachable during the duplicate check → 424 `failed_dependency` / `ld.failed_dependency`; if the settings call fails the check is skipped (`LccnUniquenessValidator.java`).
15. Settings changes are cached (5 h TTL by default); re-test after a settings toggle only after `DELETE /linked-data/admin/caches` or cache expiry (README env table, Karate LCCN features).
16. Profile settings name must be unique per user per profile → `profile_settings_name_not_unique`; blank name → `ld.error.profileSettingsNameNotBlank` (MODLD-1040, `ProfileSettingsNameUniquenessValidator.java`, en.json).
17. Saving after a concurrent ID change returns `not_found` → `ld.not_found` message; same ID → `already_exists` (errors.yml, en.json).

### Authorities
18. Authority assignment is type-checked by `POST /linked-data/authority-assignment-check`: CREATOR_OF_WORK accepts person, family, organization, jurisdiction, meeting; SUBJECT_OF_WORK accepts those plus topic, place, form, hub; DEGREE_GRANTING_INSTITUTION accepts organization only (Karate `authority-assignment-check.feature`).
19. Wrong type → toast `ld.errorAssigningAuthority.not_valid_for_target`; unmapped MARC authority → `ld.errorAssigningAuthority.unsupported_marc`; after a rejected assignment the "Assign" button must stay usable for a valid target (UILD-745, en.json).
20. Saving a description with uncontrolled authorities shows the "Uncontrolled authorities warning" modal (UILD-448/451/495, en.json).
21. Standalone Authority create/edit uses the same `/linked-data/resource` API with profiles 8–16; each type has one required, non-repeatable name field (e.g. Person "Personal name" 100/700$a, Topic "Topical term" 150/750$a) (UILD-826, MODLD-1082/1083, C1385658).
22. New authority page: "Actions" dropdown absent, X/Cancel enabled, Save buttons disabled until a change (C1385658).
23. Authority Identifiers section is repeatable, type LCCN by default (or Other); entering only an LCCN must not auto-fill the name (C1385658, UILD-838).
24. Assigned creator's label must render on first open of a Work from search results (UILD-842); assigning Creator must not blank the Instance preview panel (UILD-807).
25. Authority lookup modal: Enter submits the search; reopening the modal clears previous Search/Browse terms (UILD-499, UILD-585).
26. Authority lookup results show the source name from the Authority File (UILD-764).

### Hubs
27. LoC hub import: preview (Creator, Title, Language) → Continue → `POST /linked-data/hub?hubUri=` → Edit hub page; Cancel imports nothing (UILD-725/726, MODLD-1053, C1030061).
28. An imported LoC hub keeps source "Library of Congress, Local" after a local edit (MODLD-990).
29. Hub label = creator label + ". " + title label + ". " + language (MODLD-970).
30. Several hubs assigned as Work subjects are all kept after save (MODLD-1000); a Work whose creator came with a hub from MARC must still be editable (UILD-673).
31. Hubs support Duplicate like Work/Instance (UILD-772); new hub save buttons behave as for Work (UILD-703, C987719).

### Profiles and profile settings (workspaces)
32. Multiple named profile settings per profile per user; one can be preferred; settings are applied in the editor without page refresh (MODLD-1040, UILD-780, UILD-798, C1259767).
33. Mandatory components (those whose absence fails validation, e.g. title) cannot be moved to "Unused components" (`ld.moveUnavailable`) (UILD-728, MODLD-985).
34. Hidden components are still rendered/preserved, so their values are kept in the saved DTO (UILD-684).
35. Applying other settings while editing asks to save first (`ld.askSaveBeforeApplyingSettings`); changing profile with unsaved changes is blocked (`ld.modal.chooseProfileWarning.*`) (UILD-780/848, en.json).
36. Changing a Work profile must keep linked Instances attached (MODLD-1072); changing profile shows a data-loss warning listing affected fields (e.g. Monograph → Rare Books: URL of instance; Books → Serials: Dissertation) (UILD-577, UILD-778).
37. Default profile chosen in the change-profile modal is reflected in Manage profile settings without refresh (UILD-820).
38. A duplicated Instance keeps the original's profile (UILD-582); a duplicated Work is saved without instances (UILD-821).
39. "Statement of responsibility" is not repeatable in Monograph, Rare Books and Serials Instance profiles (MODLD-1061).

### Import / export
40. Import resources (file or URL): empty file → `ld.importFileEmptyError`; wrong type → `ld.importFileTypeError`; failure must show "Import failed", never "Import successful" (UILD-692, en.json).
41. Work type must exist in imported data, otherwise the default work type (URL import dropdown; Monographs if not chosen) is used; the downloaded CSV log has a "Type" column (MODLD-983, UILD-757, C736765).
42. Works can be RDF-imported directly, without joining an instance graph (MODLD-1109, UILD-839, C1415956).
43. Importing the same RDF via bulk and single import must not create duplicate Works (MODLD-989); instances added in UI stay searchable after a bulk import update (MODLD-1008).
44. "Export instance RDF file" calls `GET /linked-data/resource/{id}/rdf` (en.json, ModuleDescriptor).

### Inventory and other apps
45. Without Marigold capabilities Inventory hides "Edit resource in Marigold" (C651425, ui-inventory en.json `editInMarigold`).
46. LINKED_DATA instances: Overlay and Set for deletion are not available in Inventory (C503137, C566511); bulk edit rejects them (C651537).
47. MARC bib with 775/776/490 creates lightweight resources that must not appear in Inventory or Linked Data search (MODLD-980, MODLD-1071).
48. Graph ↔ MARC mappings are asserted field by field via "View MARC" (008, 020, 040, 1XX/6XX/7XX, 130/240/630 hubs, 300, 336–338 $2, 502, 773/787) (MODLD-717, MODLD-1014, MODLD-1029, Cypress LD Edit specs).
49. Admin/batch APIs (reindex, graph cleaning, caches, bulk import) return 403 `authorization_error` for users without the Marigold Administrator role (C1292045).
50. Marigold UI follows the session locale (field labels from profiles stay English) (UILD-786, C1453646, C1453647).

---

## Exact UI Texts

> From `folio-org/ui-linked-data` `translations/ui-linked-data/en.json` at master (2026-11-24) unless stated. `{x}` are placeholders. Use verbatim. `en_US.json` lags by 35 keys (e.g. `ld.profile_settings_name_not_unique`, `ld.source.localLDAuthority`, `ld.profileSettings.*` are missing there) — assert against en.json.

### Toast Messages

| Key | Text | Source |
|---|---|---|
| `ld.errorFetching` | Error fetching data | [source] |
| `ld.errorLoadingResource` | Error loading resource | [source] |
| `ld.errorLoadingProfiles` | Error loading profiles | [source] |
| `ld.errorLoadingProfileSettings` | Error loading profile settings | [source] |
| `ld.errorApplyingProfileSettings` | Error applying profile settings | [source] |
| `ld.confirmCloseRd` | Do you really want to close the resource description? All unsaved changes will be lost. | [source] |
| `ld.confirmDeleteRd` | Do you really want to delete the resource description? | [source] |
| `ld.rdSaveSuccess` | Resource created. Expect a short delay before resource is visible in FOLIO. | [source], UILD-703 |
| `ld.rdUpdateSuccess` | Resource updated. Expect a short delay before changes are visible in FOLIO. | [source] |
| `ld.rdDeleted` | Resource description deleted | [source] |
| `ld.cantSaveRd` | Cannot save the resource description | [source] |
| `ld.errorFetchingExternalResourceForPreview` | Error fetching external resource for preview | [source] |
| `ld.errorFetchingExternalResourceForEditing` | Error fetching external resource for editing | [source] |
| `ld.errorMakingApiRequest` | Error making API request | [source] |
| `ld.errorAssigningAuthority` | Error assigning authority | [source] |
| `ld.errorAssigningAuthority.not_valid_for_target` | Please select an authority with a resource type matching the edited authority field (e.g. LCNAF authority for the Creator field) | [source], UILD-745 |
| `ld.errorAssigningAuthority.unsupported_marc` | The selected authority isn't mapped to Linked Data, and can't be assigned yet. This is temporary, since MARC mapping is in progress. | [source] |
| `ld.errorImportingHub` | Error importing hub. Please try again. | [source] |
| `ld.cantLoadMarc` | Can't load MARC for this resource description | [source] |
| `ld.cantLoadProfileSettings` | Cannot load custom profile settings, using profile defaults | [source] |
| `ld.error.profileSaveAsPreferred` | Error setting up preferred profile | [source] |
| `ld.error.profileChange` | Error changing recording profile | [source] |
| `ld.error.saveProfileSettings` | Error saving profile settings | [source] |
| `ld.error.updatePreferredProfileSettings` | Error updating preferred profile settings | [source] |

### Validation / Error Messages (backend code → UI text)

| Key | Text | Source |
|---|---|---|
| `ld.required_primary_main_title` | Please add the main title in order to save. | [source] |
| `ld.required_series_title` | Please add a title to each series in order to save. | [source] |
| `ld.lccn_does_not_match_pattern` | LCCN Pattern validation enabled: Please provide an LCCN that complies with standard structure. | [source] |
| `ld.lccn_not_unique` | LCCN uniqueness enabled: The LCCN you provided already exists. Please provide a different LCCN. | [source] |
| `ld.failed_dependency` | Unable to verify if LCCN is duplicated. | [source] |
| `ld.profile_settings_name_not_unique` | A saved profile settings with this name already exists. Please choose a different name. | [source] |
| `ld.error.profileSettingsNameNotBlank` | Profile settings name should not be blank, please provide a name | [source] |
| `ld.not_found` | The resource ID has been changed by another user while you had this resource open. Please refresh your browser’s window. | [source] |
| `ld.already_exists` | The same resource ID already exists. | [source] |
| `ld.mapping` | There’s been a mapping issue. Please report this to your support team, with screenshots and a detailed account of what happened. | [source] |
| `ld.generic_bad_request` | There’s been a UI issue. Please report this to your support team, with screenshots and a detailed account of what happened. | [source] |
| `ld.server` | There’s been a server issue. Please report this to your support team, with screenshots and a detailed account of what happened. | [source] |
| `ld.appFail` | An error occurred. Please, reload the page. | [source] |
| `ld.moveUnavailable` | Required field, cannot move to unused | [source] |

### Modal / Dialog Titles and Bodies

| Key | Text | Source |
|---|---|---|
| `ld.unsavedChanges` / `ld.askSaveChangesBeforeProceeding` | Unsaved changes / Would you like to save your changes before proceeding? | [source] |
| `ld.continueWithoutSaving` | Continue without saving | [source] |
| `ld.modal.uncontrolledAuthoritiesWarning.title` | Uncontrolled authorities warning | [source] |
| `ld.modal.uncontrolledAuthoritiesWarning.body` | This description includes one or more uncontrolled authorities. (line break) Do you want to save changes? | [source] |
| `ld.duplicateImportWarn` / `ld.rdPropertiesMatchContinue` | Duplicate import warning / Properties of this resource match an existing resource. Do you want to continue? | [source] |
| `ld.modal.chooseResourceProfile.subtitle` | Choose a resource profile to get started. | [source] |
| `ld.modal.chooseResourceProfile.setAsDefault` | Set as my default | [source] |
| `ld.modal.chooseResourceProfile.typeProfile` | {type} profile | [source] |
| `ld.modal.chooseResourceProfile.workProfile` / `.instanceProfile` | Work profile / Instance profile | [source] |
| `ld.modal.chooseResourceProfile.setDefaultWorkProfile` / `.setDefaultInstanceProfile` | My default work profile / My default instance profile | [source] |
| `ld.modal.chooseResourceProfile.warningTitle` | Warning: You might lose information in the following fields: | [source], UILD-577 |
| `ld.modal.chooseProfileWarning.title` / `.message` | Unable to change profile / You must save any changes before changing the profile. | [source] |
| `ld.applyProfileSettings` / `ld.askSaveBeforeApplyingSettings` | Apply profile settings / Changing profile settings will discard unsaved changes. Save changes before proceeding? | [source] |
| `ld.unsavedProfileChanges` / `ld.unsavedProfilePrompt` | Unsaved profile changes / Would you like to save your changes before proceeding? | [source] |
| `ld.unsavedProfileNote` | Note: Only the selected components will be visible for resources using this profile. Components not in the selected section will be hidden. | [source] |
| `ld.profileComponentsDeselected` / `ld.removedProfileComponentsResult` | Profile components deselected / You have removed components from your selected list for this profile. Only the selected components will be visible for resources using this profile. Would you like to proceed? | [source] |
| `ld.visibleByDrift` | This component is shown because it is missing from your settings. Manage profile settings for this profile and save your update to clear this message. | [source] |
| `ld.importHub.header.title` | Import hub - {title} | [source] |
| `ld.marcWithTitle` | MARC - {title} | [source] |

### Import Messages

| Key | Text | Source |
|---|---|---|
| `ld.importInstances` / `ld.importHubs` | Import resources / Import hubs | [source], UILD-839 |
| `ld.importInstancesDescription` | Import resources to the Linked Data editor by uploading a file or by using a URL. | [source] |
| `ld.importHubsDescription` | Import hubs by uploading a file or by using a URL. | [source] |
| `ld.importFile` / `ld.importUrl` / `ld.importUrlLabel` | JSON File / URL / JSON file URL | [source] |
| `ld.importDefaultWorkType` | Default work type if none found | [source] |
| `ld.importFileTypeError` | You are unable to upload that file type. Please try again. | [source] |
| `ld.importFileSizeError` | Your file is too large. Please try again. | [source] |
| `ld.importFileEmptyError` | Your file is empty. Please provide a file with JSON contents. | [source] |
| `ld.importNoResourcesWarning` | Import ran successfully but no resources were imported. Review the import log or check for appropriate resource types. | [source] |
| `ld.importNoUrlError` | No URL was provided. Please try again with a URL. | [source] |
| `ld.importFileSuccess` | Your file was successfully imported and an import log file downloaded. | [source] |
| `ld.importSuccessful` / `ld.importFailed` | Import successful / Import failed | [source] |
| `ld.importFileFailure` | Your file was unable to be imported. Please check the import log downloaded to your browser for more details. | [source] |
| `ld.importTimedout` | Import failed to complete in time | [source] |

### Button / Menu Labels (key actions)

"Save & close" (`ld.saveAndClose`), "Save & keep editing" (`ld.saveAndKeepEditing`), "Save & continue" (`ld.saveAndContinue`), "Cancel", "Duplicate", "View MARC", "Export instance RDF file" (`ld.exportInstanceRdf`), "Inventory View" (`ld.inventoryView`), "New resource", "New hub", "New authority", "New instance", "Compare selected", "Manage profile settings", "Change work profile" / "Change instance profile", "Assign authority", "Assign", "Import/Assign", "Assign hub", "Import/Edit", "Change", "Create new settings", "Reset components", "Set as my preferred profile settings for this profile", "Set as my default {type} profile", "Continue", "Done", "Try again" — all from en.json. Duplicate title suffixes: "(DUPLICATE WORK)", "(DUPLICATE INSTANCE)", "(DUPLICATE HUB)", "(DUPLICATE AUTHORITY)". Sources: "Library of Congress", "Library of Congress, Local", "Local", "Local Marigold Authority", "Local MARC Authority", "Local LoC Children's Subject heading" (`ld.source.*`).

Inventory (ui-inventory en.json): `editInMarigold` "Edit resource in Marigold"; `ld.label` "Marigold".

---

## Known Gaps / Items to Verify

- [ ] Current release naming: UILD/MODLD Jira versions are labelled by number only (2.0.5 = "Trillium R1 2025 SP#1" in UILD, 2.0.4 in MODLD) — "2025" in that description looks like a typo for 2026; ui-linked-data package.json still says 2.0.4 while Jira UILD 2.0.5 is released (2026-08-29). Release label "Umbrellaleaf R2 2026" for master is inferred from MODLD NEWS v3.0.0 in progress and MODLD-1057 ("Umbrellaleaf - Authorities creation and editing").
- [ ] Admin role name: C1292045 step 1 expects role "Linked Data Administrator", while its preconditions, MODLD-1071 and stripes-testing use "Marigold Administrator" — check the env.
- [ ] C1292045 lists `linked-data-import.jobs.item.cancel.put`; ModuleDescriptor (after MODLD-1035) has `linked-data-import.jobs.item.stop.put` and PUT `…/jobs/{id}/stop` — update the case.
- [ ] mod-linked-data README documents `GET /linked-data/reindex/status`, but the ModuleDescriptor and C1292045 use `GET /linked-data/batch/status` — use the latter.
- [ ] UILD-641 toast text "Date of creation and Local resource ID number will show up after some auto-processing" is not in en.json (only `ld.placeholder.processing` "Information will be shown after processing") — don't assert the story text.
- [ ] UILD-354 asked for screen header "Search authorities" and body "Enter a search query to show results"; neither string exists in en.json (en.json has `ld.searchCreatorAuthority` "Search creator authority", `ld.searchSubjectAuthority`, `ld.chooseFilterOrEnterSearchQuery`) — verify on screen.
- [ ] Identifier status dropdown: UILD-681 (Instance) says only "canceled or invalid" and "current"; MODLD-1041 and C1385658 (Authority) show 3 values incl. "incorrect" — verify for Instance profiles.
- [ ] UILD-358 proposed validation for Creator = Other contributor with same relationship (text "Creators and Other contributors Relationship cannot be the same, please correct"); not in en.json — treat as not implemented.
- [ ] Complex Subject profile (MODLD-1098) is Closed but no profile JSON exists in `src/main/resources/profiles/` and no Complex Subject option in the New authority modal (C1395177 lists 9 types) — verify before writing cases.
- [ ] Bulk edit error text for LINKED_DATA instances ("Bulk edit of instances with source set to LINKED_DATA is not supported.") is taken from stripes-testing `cypress/support/constants.js`, not from a translation file — verify in ui-bulk-edit/mod-bulk-operations.
- [ ] Inventory UI label for source LINKED_DATA and the read-only preview "Continue" flow were not re-checked in ui-inventory translations.
- [ ] Authority Name/required error on the UI for `required_authority_name` has no `ld.required_authority_name` key in en.json — the UI message shown is unconfirmed (C1385658 says only "Validation error appeared").
- [ ] docs.folio.org has no Marigold page used here; Confluence used only for release/role context.
