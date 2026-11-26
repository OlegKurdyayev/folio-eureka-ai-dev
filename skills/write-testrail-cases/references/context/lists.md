# FOLIO Lists App (FQM) — Business Logic Context

> Built: 2026-11-24 (full refresh of the 2026-11-07 file; 2026-06-17 build + 2026-11-07 delta folded in; remediated 2026-11-25) · Release: Umbrellaleaf R2 2026 (ui-lists master after v5.0.2, ui-plugin-query-builder 3.0.3 in progress, mod-lists 4.1.0, mod-fqm-manager 4.2.0) · Builder: build-app-context/manual

> Sources (2026-11-24 refresh):
> - GitHub (master): folio-org/ui-lists (`translations/ui-lists/en.json`, `package.json`, `CHANGELOG.md`, `src/`), folio-org/ui-plugin-query-builder (`translations/ui-plugin-query-builder/en.json`, `CHANGELOG.md`), folio-org/mod-lists (`descriptors/ModuleDescriptor-template.json`, `src/main/resources/swagger.api/list.yaml` + `schemas/`, `src/main/java/org/folio/list/services/ListValidationService.java`, `exception/`, `README.md`, `NEWS.md`), folio-org/mod-fqm-manager (`descriptors/ModuleDescriptor-template.json`, `NEWS.md`)
> - Jira (anonymous REST): UILISTS, MODLISTS, MODFQMMGR, UIPQB — 840 Done/Closed stories+bugs resolved since 2024-06-01 scanned (fixVersions are module versions, so scoped by resolution date); ~70 read in detail with comments
> - folio-integration-tests: `mod-lists/`, `mod-fqm-manager/`, `edge-fqm/` Karate projects · stripes-testing: `cypress/e2e/lists/`, `cypress/e2e/consortia/lists/`, `cypress/support/fragments/lists/`
> - Confluence: "Lists App permissions" (572948576), "Lists App: Cross-tenant Queries [ECS only]" (599916623), "Returning null/empty values in Lists / FQM" (932380775), "Lists app - Trillium - Notes on functionality" (2073591809), "Lists app - Umbrellaleaf - Notes on functionality" (2391769089), "Umbrellaleaf - MARC Querying in Lists app" (2344353798)
> - docs.folio.org source: `content/en/docs/Lists/_index.md`
> - TestRail project 14 / suite 21: PARTIAL. Section 29149 had no MCP access on 2026-11-24; on 2026-11-25 the MCP was reachable and these case bodies were read: C1045968, C1632042, C1505043 (section lists: 29149, 29154, 34769 listed by title only). Other case IDs come from earlier builds and stripes-testing spec titles. See Known Gaps.
> Strings marked [en.json `key`] are verbatim from `translations/<module>/en.json` on master. Strings marked [from cases] come from TestRail cases (earlier builds).

---

## What is Lists

Lists is the FOLIO application for building, saving, refreshing, and exporting query-based record sets across many record types. It is powered by FQM (FOLIO Query Machine, mod-fqm-manager), which consolidates data from multiple modules for cross-application querying; the query UI is the shared plugin ui-plugin-query-builder. Lists are persisted artifacts (not live views) — the record set is static until refreshed, but field values shown for each record are read at view time (docs `Lists/_index.md` → "Update a record set in a list"). Lists are commonly used as a selection step before bulk actions such as Bulk Edit. Each list can hold up to 1.25 million records (`MAX_LIST_SIZE` default 1250000, mod-lists `README.md`; docs).

---

## Key Terms

| Term | Definition |
|---|---|
| FQM | FOLIO Query Machine query engine (mod-fqm-manager) used by Lists |
| List | Saved query definition (`fqlQuery`) plus a refreshable record set (mod-lists `ListDTO.json`) |
| Record type / entity type (ET) | Entity queried by the list (Items, Users, Invoices, Authority, …); simple, composite, or custom ET |
| Custom entity type | ET created via API (`POST /entity-types/custom`) that joins existing sources; usable in the Lists UI (Trillium notes, MODFQMMGR-771) |
| Query Builder | Full-screen layer (UIPQB-65) used to define Field / Operator / Value rows joined with `AND` |
| Record set | Results produced by the latest successful refresh (`successRefresh.recordsCount`) |
| Canned list | System-generated list (`isCanned=true`); shown with Source `System` (UILISTS-263); e.g. `Missing items`, `Inactive patrons with open loans`, expired patron loans list (stripes-testing `fragments/lists/lists.js`) |
| Cross-tenant query | ECS only: query from the Central tenant returning data from several tenants (Confluence 599916623) |
| Content (entity-type) permissions | Module permissions the user needs to view the underlying records; required in addition to Lists app capabilities (Confluence 572948576) |

---

## Record Hierarchy / Architecture

Lists UI (ui-lists) -> Query Builder plugin (ui-plugin-query-builder) -> mod-lists (`/lists`) -> mod-fqm-manager (`/query`, `/entity-types`) -> source modules' DB views

- List (mod-lists): `name`, `description`, `entityTypeId`, `fqlQuery`, `fields`, `isActive`, `isPrivate`, `isCanned`, `isDeleted`, `version`, `createdBy/updatedBy(+Username)`, `successRefresh` / `inProgressRefresh` / `failedRefresh` (`ListDTO.json`)
- List versions: every create/update stores a historic version (`GET /lists/{id}/versions`) (`ListService.java`; Karate `versions.feature`)
- Refresh (`ListRefreshDTO`): `status` IN_PROGRESS / SUCCESS / FAILED / CANCELLED, `recordsCount`, `contentVersion`, `error`
- Export (`ListExportDTO`): `status` IN_PROGRESS / SUCCESS / FAILED / CANCELLED; CSV staged in S3/MinIO

---

## Lifecycle / Workflow

| Status | Description | Source |
|---|---|---|
| Active | Query + resulting record set; can be refreshed and exported | [en.json `create-list.main.list-status.tooltip`] |
| Inactive | Can have a query, but no record set; refresh and export blocked | [en.json `create-list.main.list-status.tooltip`], `ListValidationService.java` |
| Deleted (API only) | Soft delete: `isDeleted=true`, contents removed; hidden from `GET /lists` unless `includeDeleted=true`; `GET /lists/{id}` returns 404 | `ListService.deleteListAndContents`, `list.yaml`, Karate `delete.feature` (FAT-11792) |

Transitions:
- Active -> Inactive (edit): list contents are deleted and `successRefresh` cleared (`ListService.java` update path); UI warns [en.json `warning.inactive-status`].
- Inactive -> Active (edit): no automatic refresh; detail page shows [en.json `callout.list.active`] until the user refreshes.
- Saving a new/changed query with status Active runs the query and imports the results (`queryId` on POST/PUT, `ListService.java`); the detail page shows `Query in progress` until it finishes (UILISTS-276).
- Refresh / export run asynchronously; refresh can be cancelled (`DELETE /lists/{id}/refresh`), export can be cancelled (`POST /lists/{id}/exports/{exportId}/cancel`).

Primary action flow: Create (+ Test query) -> Run query & save -> (Refresh | Export | Duplicate | Edit | Delete)

---

## API Surface (mod-lists `lists` 2.0, mod-fqm-manager `entity-types` 2.1 / `fqm-query` 2.1)

> From `descriptors/ModuleDescriptor-template.json` (master). ui-lists requires `lists` 1.0 2.0, `fqm-query` 2.0, `entity-types` 2.0 (`package.json`).

| Method + path | permissionsRequired |
|---|---|
| GET /lists | `lists.collection.get` |
| POST /lists | `lists.collection.post` |
| GET /lists/{id} | `lists.item.get` |
| PUT /lists/{id} | `lists.item.update` |
| DELETE /lists/{id} | `lists.item.delete`, `fqm.entityTypes.used-by.item.patch` |
| POST /lists/{id}/refresh | `lists.item.post` |
| DELETE /lists/{id}/refresh | `lists.item.refresh.cancel` |
| GET /lists/{id}/contents | `lists.item.contents.get` |
| POST /lists/{id}/exports | `lists.item.export.post` |
| GET /lists/{id}/exports/{exportId} | `lists.item.export.get` |
| GET /lists/{id}/exports/{exportId}/download | `lists.item.export.download.get` |
| POST /lists/{id}/exports/{exportId}/cancel | `lists.item.export.cancel` |
| GET /lists/{id}/versions, /versions/{versionNumber} | `lists.item.versions.collection.get`, `lists.item.versions.item.get` |
| GET /lists/configuration | `lists.configuration.get` (returns `maxListSize`) |
| GET /lists/related-users?role=create\|update | `lists.related-users.collection.get` (MODLISTS-277) |
| GET /entity-types, /entity-types/{id} | `fqm.entityTypes.collection.get`, `fqm.entityTypes.item.get` |
| GET /entity-types/{id}/field-values | `fqm.entityTypes.item.columnValues.get` |
| POST/GET/PUT/DELETE /entity-types/custom[/{id}] | `fqm.entityTypes.custom.*` |
| POST /query, GET /query/{id}, GET /query | `fqm.query.async.post`, `fqm.query.async.results.query.get`, `fqm.query.sync.get` |

`GET /lists` query params (`list.yaml`): `ids`, `entityTypeIds`, `offset`, `size`, `search` (name/description), `active`, `private`, `canned` (only `true`/`false`; omit for all — an empty value returns 500 `unhandled.error`, MODLISTS-243 comment), `createdBy`, `updatedBy` (multi-value), `includeDeleted`, `updatedAsOf`, `sortBy` (`name`, `updatedDate`, `recordsCount`), `sortOrder` (`asc`, `desc`). `POST /lists` requires `name`, `entityTypeId`, `isActive`, `isPrivate` (`ListRequestDTO.json`); missing ones return 400 (Karate `create.feature`). `PUT` requires `version` (optimistic lock).

Error codes: `<action>-<code>` where action is `create`/`read`/`update`/`delete`/`refresh`/`cancel_refresh`/`export`/`cancel_export` (`AbstractListException.getErrorCode`). Codes and HTTP status: `list.not.found` 404, `entity.not.found` 404, `entity.type.restricted` 403, `list.is.private` 401, `list.is.inactive` 400, `list.is.canned` 400, `list.refresh.in.progress` 400, `export.in.progress` 400, `list.not.refreshing` 400, `export.not.in.progress` 404, `list.missing.query` 400, `max.list.size.exceeded` 400, `optimistic.lock.exception` 409, `fql.query.invalid` 400, `list.is.cross-tenant.and.not.private` 400, `list.contents.request.failed` 409, `version.not.found` 404 (mod-lists `exception/*.java`). The UI maps each code to an en.json key of the same name (e.g. `refresh-list.is.inactive`).

---

## Main Sections / UI Structure

### Panes
- `Search & filter` [en.json `filterPane.title`] (renamed from `Filter`, UILISTS-252; shortcut label [en.json `commands-label.go-to-filter-pane`] `Go to Search & filter pane`) — search box labelled `Search lists` [en.json `lists.searchInputLabel`] above the filter accordions
- `Lists` [en.json `mainPane.title`], subtitle [en.json `mainPane.subTitle`] `No records found` / `1 record found` / `# records found`; `Query in progress` [en.json `mainPane.queryInProgress`]
- New list page: `New list` [en.json `title.createList`]; Duplicate page: `Duplicate List: {listName}` [en.json `lists.copy.title`]; Edit page: `Edit {listName}` [en.json `lists.edit.title`]
- Build query layer: trigger `Build query` [en.json UIPQB `trigger`], full-screen (UIPQB-65); header shows the list name and `Record type:` [en.json UIPQB `modal.entityType`] (UILISTS-245)

Landing-page table columns: `List name`, `Record type`, `Records`, `Status`, `Source`, `Last updated`, `Visibility` [en.json `lists.columns.*`]. Page size 100 (`src/constants/base.ts` `LISTS_DEFAULT_PAGE_AMOUNT`; C411697/C411698).

No-results messages: [en.json `mainPane.noResults`] `No results found for "{searchTerm}". Please check your spelling and filters.` / [en.json `mainPane.noResultsFilters`] `No results found. Please check your spelling and filters.` (filters only).

### Accordions / filters
- `Status` (`Active`/`Inactive`), `Visibility` (`Shared`/`Private`), `Record types`, `Source` (`System` / `User generated`), `Created by`, `Updated by` [en.json `filter-label.*`, `lists.item.*`]
- `Created by` / `Updated by`: multi-select dropdowns of users who created/updated at least one list, alphabetical, collapsed by default (UILISTS-281; replaced the `Find user` plugin of UILISTS-253)
- List detail: `List information` accordion [en.json `accordion.title.list-information`]; query accordion `Query: {query}` with `Show more` / `Show less` [en.json UIPQB `viewer.accordion.title.query*`]

Sorting by `List name`, `Last updated`, `Records` (asc/desc carets) (UILISTS-254, MODLISTS-68). Search term, filters and sort persist across view/add/edit/duplicate/delete, page reload and app navigation (UILISTS-279, C1538701/C1538703); `Reset all` clears search and filters and returns to the default sort, but does not reset a user-chosen sort per UILISTS-254 refinement — see Known Gaps. The create/edit list form no longer has `Collapse all` / `Expand all` (UILISTS-255; keys removed from en.json) — do not assert.

### Key Navigation Paths
- Lists -> `New` -> fill `List information` -> `Build query` -> `Test query` -> `Run query & save` [docs; en.json]
- Lists -> list details -> `Actions` -> `Refresh list` / `Cancel refresh` / `Edit list` / `Duplicate list` / `Delete list` / `Export all columns (CSV)` / `Export selected columns (CSV)` / `Cancel export` / `Show columns` [en.json `pane.dropdown.*`]
- Lists -> list details -> `Actions` -> `Edit list` -> `Edit query` [en.json `list.modal.edit-query`]
- Lists -> `Search & filter` pane -> `Status` / `Visibility` / `Record types` / `Source` / `Created by` / `Updated by`
- App context menu: `Lists app home`, `Keyboard shortcuts` [en.json `app-menu.*`] (UILISTS-138, C535523)

---

## Exact UI Texts

> Use these verbatim in expected results. Never paraphrase. All strings re-verified 2026-11-24 against ui-lists / ui-plugin-query-builder `en.json` on master. `en_US.json` (Lokalise copy) differs only by still holding the removed `create-list.main.collapse-all`/`expand-all` and `find-user.*` keys.

### Toast / Callout Messages

| Event | Toast text | Key |
|---|---|---|
| List saved | `List <strong>{listName}</strong> saved.` | `callout.list.save.success` |
| List save error | `Error: <strong>{listName}</strong> was not saved.` | `callout.list.save.error` |
| List created | `List <strong>{listName}</strong> was created. Reload to see changes. Note: the list may not appear based on filters.` | `callout.list.created` |
| List deleted | `List <strong>{listName}</strong> deleted.` | `callout.list.delete.success` |
| Delete error | `Error: <strong>{listName}</strong> was not deleted.` | `callout.list.delete.error` |
| Refresh complete | `<strong>Refresh complete</strong> with <strong>{count}</strong> records: ` + link `View updated list` | `status-toast.success.refresh-complete`, `status-toast.success.link` |
| Refresh failed | `The list <strong>{listName}</strong> was not refreshed. Please try again.` | `callout.list.refresh.error` |
| List is active (needs refresh) | `<strong>{listName}</strong> is active. Refresh <strong>{listName}</strong> to see list contents` | `callout.list.active` |
| CSV export begin | `Export of <strong>{listName}</strong> is being generated. This may take some time for larger lists.` | `callout.list.csv-export.begin` |
| CSV export success | `List <strong>{listName}</strong> was successfully exported to CSV.` | `callout.list.csv-export.success` |
| CSV export failed | `Error: Export for list <strong>{listName}</strong> failed.` | `callout.list.csv-export.error` |
| CSV export cancelled | `The export for <strong>{listName}</strong> was successfully cancelled` (no final period) | `callout.list.csv-export.cancel` |
| Refresh cancelled | `The refresh for <strong>{listName}</strong> was successfully cancelled.` | `cancel-refresh.success` |
| Refresh in progress (detail page) | `List refresh in progress` | `refresh.status.in-progress` |

### Modal / Dialog Messages

| Modal | Heading / message | Key |
|---|---|---|
| Delete list | `Delete list` / `Are you sure you want to delete the list <strong>{listName}</strong>?` (button `Delete`) | `list.modal.delete-list`, `list.modal.confirm-message`, `list.modal.delete` |
| Unsaved changes | `Are you sure?` / `There are unsaved changes` (buttons `Keep editing`, `Close without saving`) | `list.model.sure-heading`, `list.modal.confirm-cancel-message`, `list.modal.keep-edit`, `list.modal.cancel-edit` |
| Inactive-status warning | `Warning: making status inactive will clear list contents.` | `warning.inactive-status` |
| Cross-tenant tip (ECS) | `<strong>Lists with cross-tenant query can only be set to Private.</strong>` | `create-list.cross-tenant-tip` |
| Cross-tenant indicator (ECS) | `This list may contain records from multiple tenants.` | `cross-tenant-message` |
| Visibility tooltip | `<strong>Shared</strong> lists can be accessed by all users with permissions to the Lists app in this organization. <strong>Private</strong> lists can only be accessed by the user logged in.` | `create-list.main.list-visibility.tooltip` |

### Query Builder texts (ui-plugin-query-builder en.json)

| Element | Text | Key |
|---|---|---|
| Buttons | `Test query`, `Run query`, `Cancel`; in Lists: `Run query & save` | `modal.test`, `modal.run`, `modal.cancel`; ui-lists `list.modal.run-query-and-save` |
| Running | `Compiling results...`, `Executing test query...`, `Counting results...` | `modal.run.inProgress`, `modal.preview.queryInProgress`, `modal.preview.countingInProgress` |
| Preview | `Query returns {total} records. Previewing the first {limit} records:` (ICU plural; `no records` when 0) | `modal.preview.title`, `modal.preview.title.empty` |
| Over max size | `This query returns more records than the maximum allowed; please modify the query to reduce the results.` | `modal.banner.limit` |
| Columns | `Boolean`, `Field`, `Operator`, `Value`, `Actions` | `columns.*` |
| Placeholders | `Select field`, `Select operator`, `Select value`, `Search fields`, `Separate multiple values with a comma.` | `control.*` |
| Organization lookup | `Organization look-up`, `Donor organization look-up`; `No matches! Use the “Organization look-up” below to add organizations` | `control.search.button.*`, `noOptionsAvailable.*` |
| Empty list | `The list contains no items` | `result.emptyMessage` |
| Deleted record row | `Deleted`, cells `—`, tooltip `This record has been deleted; refresh the list to remove it from the results.` | `viewer.deletedRecord.*` (UIPQB-72, C957375) |
| Deleted field | `{value} in your query is no longer available. Please revise your query.` | `warning.deletedField` |
| ET changed | `The record type used by this list has changed; please refresh the list, and if needed, modify the query.` | `error.needsRefresh` |
| ET broken | `The record type used by this list is currently unable to be used; please verify its definition.` | `error.invalidEntity` |
| Generic | `Something went wrong`, `An error occurred.` | `error.sww`, `error.occurredMessage` |

### Button / Field Labels
Actions: `New`, `Actions`, `Refresh list`, `Cancel refresh`, `Delete list`, `Edit list`, `Duplicate list`, `Cancel export`, `Show columns`, `Export all columns (CSV)`, `Export selected columns (CSV)`, `Run query & save`, `Edit query`, `Save`, `Cancel` [en.json `paneHeader.button.*`, `pane.dropdown.*`, `button.*`]. Fields: `List name`, `Description`, `Visibility` (`Shared`/`Private`), `Status` (`Active`/`Inactive`), `Record type` (placeholder `Select record type`), `Source` [en.json `create-list.main.*`, `list.info.*`]. Show columns search placeholder `Search column name` [en.json `pane.dropdown.search.placeholder`]. Required-field validation: `Please fill this in to continue` [en.json `form.field.empty-error`]. Duplicate default name: `<source name> - copy` (`useCopyListState.ts` + [en.json `lists.copy.name-postfix`]). Record-type disambiguation: `{label} (built-in)` / `{label} (custom)` [en.json `record-types.label.deduplication.*`] (UILISTS-243).

### Error Messages (blocked operations) — [en.json, key in table]

| Condition | Message text | Key |
|---|---|---|
| Delete during refresh | `Error: <strong>{listName}</strong> was not deleted. Lists can't be deleted while a refresh is in progress.` | `delete-list.refresh.in.progress` |
| Delete during export | `Error: <strong>{listName}</strong> was not deleted. Lists can't be deleted while an export is in progress.` | `delete-export.in.progress` |
| Update during refresh | `Error: changes to <strong>{listName}</strong> were not saved. Lists can't be updated while a refresh is in progress.` | `update-list.refresh.in.progress` |
| Update during export | `Error: changes to <strong>{listName}</strong> were not saved. Lists can't be updated while an export is in progress.` | `update-export.in.progress` |
| Optimistic lock conflict | `Error: someone else modified <strong>{listName}</strong>. Reload the page to view the latest version of this list.` | `update-optimistic.lock.exception` |
| Refresh while inactive | `Error: <strong>{listName}</strong> was not refreshed because it is inactive. Set the list status to active to refresh the list.` | `refresh-list.is.inactive` |
| Refresh already running | `Error: refresh for <strong>{listName}</strong> failed because a refresh for this list is already in progress.` | `refresh-list.refresh.in.progress` |
| Refresh during export | `Error: <strong>{listName}</strong> was not refreshed. Lists can't be refreshed while an export is in progress.` | `refresh-export.in.progress` |
| Refresh missing query | `Error: <strong>{listName}</strong> was not refreshed. Add a query before refreshing the list.` | `refresh-list.missing.query` |
| Refresh exceeds max size | `Error: refresh for <strong>{listName}</strong> failed because the refreshed list contains too many records.` | `refresh-max.list.size.exceeded` |
| Refresh after list made private by another user | `Error: refresh of <strong>{listName}</strong> failed. Someone else modified this list and you no longer have access to it.` | `refresh-list.is.private` |
| Export during refresh | `Error: <strong>{listName}</strong> was not exported. Lists can't be exported while a refresh is in progress.` | `export-list.refresh.in.progress` |
| Export already running (same user) | `Error: export of <strong>{listName}</strong> failed because an export for this list is already in progress.` | `export-export.in.progress` |
| Export of inactive list | `Error: export of <strong>{listName}</strong> failed. Verify the list is active and try to export the list again.` | `export-list.is.inactive` |
| Export after list made private by another user | `Error: export of <strong>{listName}</strong> failed. Someone else modified this list and you no longer have access to it.` | `export-list.is.private` |
| Cancel export not running | `Error: the export for <strong>{listName}</strong> was not cancelled as the export is not in progress. The export may have already completed.` | `cancel_export-export.not.in.progress` |
| Cancel refresh not running | `Error: the refresh for <strong>{listName}</strong> was not cancelled. Verify a refresh is in progress and try again.` | `cancel_refresh-list.not.refreshing` |
| Delete after list made private | `Error: the list <strong>{listName}</strong> was not deleted. Someone else modified this list and you no longer have access to it` (no final period) | `delete-list.is.private` |
| List / ET not found (any action) | e.g. `Error: changes to <strong>{listName}</strong> were not saved because the list was not found. Verify the list location and try again.` | `*-list.not.found`, `*-entity.not.found` |
| Query invalid (deleted field) | `Unable to complete action. Query is no longer valid due to a deleted field(s). Please edit the list query or delete the list.` | `create-fql.query.invalid`, `update-fql.query.invalid` |
| Detail page: no ET permission | `You do not have permission to view this list` | `error-component.read-entity.type.restricted` |
| Detail page: private list of another user | `You don't have permission to view this list` | `error-component.read-list.is.private` |
| Detail page: list missing | `List not found` | `error-component.read-list.not.found` |
| Detail page: ET gone | `This list's entity type is no longer available` | `error-component.read-entity.not.found` |
| App-level: no ET permissions at all | `You do not have the required permissions to use the Lists app` | `catastrophic-error.missing-all-entity-type-permissions.heading` |
| App-level: broken custom ET source | `A broken record type is preventing access to the Lists app` | `catastrophic-error.entity.type.source.missing.heading` |
| Shortcut without permission | `User does not have permission to take this action` / `This action is unavailable` | `commands-error.permission`, `commands-error.unavailable` |

---

## Permission / Entity-Type Access Gating

- **Zero Lists capability sets → no Lists app**: absent from the Apps dropdown; `/lists` shows `You don't have permission to view this app/record` (generic FOLIO page, not a Lists string) (C476848, C476851).
- **Lists capability but no entity-type permissions at all** → the app opens with the full-page message [en.json `catastrophic-error.missing-all-entity-type-permissions.heading`] (MODLISTS-112 comments, MODFQMMGR-658).
- **Only permitted ETs are offered and listed**: `GET /lists` returns only lists whose ET the user may access; the `Record type` dropdown shows only permitted ETs (`ListService.getAllLists` uses the user's ET summary; MODLISTS-112). A user needs **all** "get" permissions of an ET (composite ETs need every source's permission, e.g. Loans needs user, item, holdings and loan permissions); partial permissions give no partial access (UILISTS-128; Confluence 572948576).
- **ET permission is re-checked on every read/refresh/export/delete** (`assertUserHasEntityTypePermissions` in `ListValidationService.java`): removing the ET permission after list creation blocks that list's URL even with full Lists Admin (C476851). The detail page then shows [en.json `error-component.read-entity.type.restricted`] (`You do not have permission to view this list`); earlier cases recorded the generic `You don't have permission to view this app/record` — see Known Gaps.
- **Private lists** are visible and operable only by their current owner = `updatedBy` if set, else `createdBy` (`assertSharedOrOwnedByUser`); other users get 401 `list.is.private` (Karate `access-control.feature`, C411710).

## Queryable vs. Displayable Fields (C451499, docs `Lists/_index.md`)

Some array-type fields cannot be used in the Query Builder but ARE available in `Show columns` and in CSV export (e.g. `User — Department UUIDs`; per-ET companions for POL, Items, Holdings, Instances — Cypress `verify-non-queryable-array-fields-in-export-and-columns*.cy.js`). The docs table per record type has columns "Can use to build query in UI?" / "Option to show as column heading in UI?" — use it as the reference for a given field. `User — Address` was made queryable in Trillium (MODFQMMGR-1043; C1045968 confirms sub-fields `Address — City/Country/Line 1/Line 2/Postal code/Primary address/Region/Type`, and `Show columns` → `User — Address`) although C451499 and docs.folio.org still list it as not queryable — see Known Gaps.

## Query Builder — Operator/Value Interaction Rules

- **Operators per data type** (labels [en.json UIPQB `operators.*`]): `equals`, `not equal to`, `greater than`, `less than`, `greater than or equal to`, `less than or equal to`, `in`, `not in`, `contains`, `not contains`, `starts with`, `is null/empty`. The user-friendly query uses symbols `==`, `!=`, `>`, `<`, `>=`, `<=`, words `in`, `not in`, `contains`, `not contains`, `starts with`, `is null/empty`, and `AND` between rows [en.json UIPQB `operators.symbol.*`, `operators.boolean.$and`]. Only `AND` is supported (no `OR`; Confluence 2344353798 UAT feedback).
- **UUID fields** all use `equals`, `in`, `not in`, `is null/empty` and display the UUID (not a name) after save (UIPQB-295, MODFQMMGR-1222; Confluence 2391769089).
- **Legacy array operators** `contains any` / `not contains any` / `contains all` / `not contains all` were migrated to `in` / `not in` / `equals`+AND / `not equal to`+AND and removed (MODFQMMGR-898/899/900/901/889, Trillium). Do not use them in new cases.
- **`in` on free-text strings**: only for flagged fields — user barcode, item barcode, Instance/Holdings/Item HRID; `not in` is NOT offered for them (MODFQMMGR-1081).
- **Value retention on operator switch**: switching among `equals`, `not equal to`, `starts with`, `contains` keeps a free-text value (UIPQB-250); `equals`↔`not equal to`, `contains`↔`starts with`, `in`↔`not in` keep the value (UIPQB-180; C651429, C651430; Cypress `verify-value-remains-*.cy.js`); `equals`→`in` keeps the first value (Cypress `verify-operator-change-keeps-first-value.cy.js`). Switching **to or from `is null/empty`** always resets the value (UIPQB-250; Cypress `verify-value-resets-when-changing-to-is-null-empty.cy.js`).
- **No `Undefined` artifacts**: the user-friendly query never shows `undefined` after an operator switch or when re-editing a saved query; the value shows its label (e.g. `English`), not its code (UIPQB-242, UIPQB-260, C1259783).
- **Value-field UI**: single-value operators show `Select value` (UIPQB-268) with a filterable dropdown (UIPQB-259); `contains`/`starts with` and filtering are case-insensitive (MODFQMMGR-994, UIPQB-147). The `Field` dropdown uses fuzzy matching and matches `-` to `—` (`loan status` finds `Loan — Status name`) (UIPQB-281).
- **Rows**: any row can be removed with the trash icon, including the first, as long as more than one row exists; a single remaining row cannot be deleted (UIPQB-182).
- **Negation operators include null/empty**: `not equal to` (string, integer, date, boolean, array of strings) and `not in` (string, UUID, arrays) return records whose field is null or empty (MODFQMMGR-753/754, Confluence 932380775; C740199, C740217/C740218/C740220).
- **`is null/empty` on arrays**: `True` matches if any element is empty; `False` matches only if all elements are non-empty (Confluence 932380775).
- **Values are shown exactly as stored** — extra spaces are not collapsed in the results table, and a copied value finds the record (UIPQB-303, C1574533).
- **Dates** are converted using the tenant time zone, not the user's (UIPQB-126, UIPQB-205). C1632042 (UIPQB-304): with tenant time zone e.g. Europe/Minsk (UTC+3), a user with `expirationDate` `2026-05-01T23:59:59.999+00:00` is found by `User — Expiration date equals 5/2/2026` and shown as 5/2/2026 in the plugin, the list details and the CSV (`2026-05-02`); after changing the tenant time zone to America/Phoenix the same user is found with 5/1/2026 (clear local storage after changing the zone). Query area shows `(users.expiration_date == 05/02/2026)`.

## Query Builder — MARC field querying (Umbrellaleaf; UIPQB-278/287/277, MODFQMMGR-1141/1142/1143/1188/1190/1191/1211/1224)

- Record types `Instances with MARC bibliographic` and `Authority`: field option `MARC field` [en.json UIPQB `marc.fieldOption`] with inputs `Tag`, `Indicator 1`, `Indicator 2`, `Subfield`. Tag is required (3 digits); the operator list appears after a tag is entered; `\` means a blank indicator (Confluence 2344353798).
- Validation: `Enter a 3-digit tag`, `Enter a-z or 0-9` (subfield), `Enter a-z, 0-9, or \ for blank` (indicators) [en.json UIPQB `marc.validation.*`].
- Result columns are labelled `MARC <tag/indicator/subfield>`; each MARC condition joined with AND gets its own column (Confluence 2344353798).
- Leader (LDR) positions and control fields 006/007/008 can be queried as the whole field, a single position, or a range (C1599856…C1599869; MODFQMMGR-1190/1224). `not equal to`/`not in` on leader positions also match FOLIO instances without a MARC record (MODFQMMGR-1190 comment).
- An empty MARC field must not be dropped from results (MODFQMMGR-1215); `contains` works on repeatable fields (MODFQMMGR-1208); removing a MARC condition row must not leave its Tag/Indicator values in the next row (UIPQB-299).

## Historical Migration Note: Regex Operators Removed (C831960)

Lists created before Sunflower stored `starts with`/`contains` as `$regex`. These are migrated: `GET /lists/{id}` `fqlQuery` shows `$contains` / `$starts_with`, never `$regex` (MODFQMMGR-614; Karate `migration.feature` "Migrate V19 queries with $regex operators"). Assert on `$contains`/`$starts_with`.

## Concurrent Edit / Multi-User Behavior (C411739, C411740, C411741, C411765–C411776, C411814–C411816)

- Two users editing the same list: the first Save succeeds (`List <strong>{listName}</strong> saved.`); the second fails with [en.json `update-optimistic.lock.exception`] because the `version` no longer matches (409, `assertListVersionMatched`). Test with two real sessions.
- Edit, delete and refresh are blocked for **any** user while the list is refreshing or exporting (`assertListNotRefreshing`, `assertListNotExporting`).
- Export: the **same** user cannot start a second export of a list while theirs is running; another user can (`assertUserNotExporting` checks the current user only).
- If another user makes a shared list private, the first user's refresh/export/delete fails with the `*-list.is.private` messages (C411816).

## Export Details (C451513, MODLISTS-232, Trillium notes)

- Two export actions: `Export all columns (CSV)` and `Export selected columns (CSV)` (selected = currently shown columns) (UILISTS-157, UILISTS-201). Export is not available from the Edit page (UILISTS-235); export actions are disabled for inactive lists (C411840) and hidden without the Export capability.
- Since Trillium, CSV column headers use the same labels as the UI (Confluence 2073591809 "Improve column headings in export to CSV"); C451513 (older) allowed snake_case headers like `loans.status_name` — see Known Gaps.
- ID columns are included only when selected (MODLISTS-232, closed 2025-11-21); C451513 (older) allowed unselected ID columns to appear — see Known Gaps.
- Record count in the file must match the list count exactly, including >10,000 rows (C451513); duplicate labels are disambiguated with the code, e.g. `German [de]` / `German [ger]` (MODLISTS-249); values are localized to the user's language (MODLISTS-265).
- Toasts for a failed export stay on screen (UILISTS-220).

## Custom entity types (Trillium+, API-created)

- Created via `POST /entity-types/custom`; validation rejects missing `shared`, null/invalid `owner`, `isCustom` null/false, sources without `targetId` (Karate `custom-entity-types.feature`).
- A custom ET in use by a list cannot be deleted: `DELETE /entity-types/custom/{id}` returns 409 "Cannot delete custom entity type because it is used by the following consumers: …" (MODFQMMGR-959; mod-lists updates `usedBy` on list create/delete, MODLISTS-225).
- Lists whose ET was deleted disappear from the landing page and their URL shows [en.json `error-component.read-entity.not.found`]; the app keeps working (UILISTS-239). A custom ET with zero ID sources returns 409 "Failed to retrieve list contents … The upstream data schema is invalid." (MODLISTS-223). A custom ET referencing a missing source shows [en.json `catastrophic-error.entity.type.source.missing.heading`] (MODFQMMGR-1018).

---

## Status Values

### List Status
`Active`, `Inactive` (UI); soft-delete flag `isDeleted` (API only).

### Refresh / Export status (API)
`IN_PROGRESS`, `SUCCESS`, `FAILED`, `CANCELLED` (`ListRefreshDTO.json`, `ListExportDTO.json`).

### Source
`System` (canned) / `User generated` [en.json `lists.item.system`, `lists.item.user-generated`].

### Visibility
`Shared` / `Private` [en.json `lists.item.shared`, `lists.item.private`]; `Compiling` [en.json `lists.item.compiling`] is shown in the Records column while a query runs (UILISTS-276).

---

## Capability Sets (Eureka)

> Source: docs.folio.org `Lists/_index.md` (Capabilities table), Confluence "Lists App permissions" (572948576), ui-lists `package.json` permissionSets, stripes-testing `support/dictionary/capabilitySets.js`. Type for all Lists capability sets: `Settings`.

| Functionality | Capability set (Type Settings) | Legacy permission (reference only) | Name confirmed by |
|---|---|---|---|
| View lists | `Settings - Module Lists Enabled - View` | `module.lists.enabled` — Lists (Enable): Can view lists | docs.folio.org `Lists/_index.md` (resource `Module Lists Enabled`, view); not in the Cypress dictionary |
| Create, edit, refresh | `Settings - Module Lists Refresh - View` | `module.lists.refresh` — Lists (Edit) | Cypress `capabilitySets.js` `moduleListsRefreshView`; docs |
| Create, edit, refresh, export | `Settings - Module Lists Export - View` | `module.lists.export` — Lists (Export) | Cypress `moduleListsExportView`; docs |
| Create, edit, refresh, delete | `Settings - Module Lists - Delete` | `module.lists.delete` — Lists (Delete) | docs.folio.org only (resource `Module Lists`, delete); not in the Cypress dictionary |
| All (Admin) | `Settings - Module Lists - Manage` | `module.lists.all` — Lists (Admin) | Cypress `moduleListsManage`; C1632042 precondition `Module Lists - Manage` |

What each level shows in `Actions` (Cypress `cypress/e2e/lists/permissions/*.cy.js`):
- View: no `New` button; `Actions` has no Refresh/Edit/Duplicate/Delete/Export (`lists-view-permissions.cy.js`).
- Edit (Refresh): Refresh, Edit, Duplicate; no Delete, no Export (`lists-edit-permissions.cy.js`).
- Delete: Refresh, Edit, Duplicate, Delete; no Export (`lists-delete-permissions.cy.js`).
- Export: Refresh, Edit, Duplicate, both Export actions; no Delete (`lists-export-permissions.cy.js`).
- Admin: all actions (`lists-all-permissions.cy.js`, C411693/C411694).

Content permissions: add the record-type's view capability sets (e.g. `UI-Users - View`, as used in C1632042 preconditions; exact capability set names per ET: capability set name not confirmed — verify) — the user needs all of them for the chosen ET (Confluence 572948576; C476848).

---

## Common Verification Patterns

### MARC Authority as a queryable Record type [C1322605]
`Authority` is a selectable `Record type`; fields include `Authority — See also from reference — Tracing` / `— Tracing type`, combinable with `starts with`, `contains`, `equals`, `not equal to`, `in` (multi-value picklist e.g. `Personal name`, `Corporate name`). `Test query` before `Run query & save` as for any record type.

### Actions menu availability on list details (C411840, permission specs)
Action: Open a list and click `Actions`.
Expected (Admin, Active list with records): `Refresh list`, `Edit list`, `Duplicate list`, `Delete list`, `Export all columns (CSV)`, `Export selected columns (CSV)`, `Show columns` are enabled.
Expected (Inactive list): `Refresh list` and both export actions disabled; `Edit list`, `Duplicate list`, `Delete list` enabled; records shows `No records found`.
Expected (canned list): `Edit list` disabled (C411731; C411769 checks the same for the delete flow); refresh, duplicate and export available (Cypress `editlist-canned.cy.js`, `delete-list/canned-lists.cy.js`, `export-canned-lists.cy.js`, C411810). The API rejects update/delete of canned lists with 400 `list.is.canned`.

### Duplicate list prepopulation (Cypress `duplicate-list/*.cy.js`)
Action: `Actions` -> `Duplicate list`.
Expected: page `Duplicate List: <name>`; `List name` = `<name> - copy`; description, status, visibility and query copied; `Save` creates an independent list; the source list is unchanged. Canned lists can be duplicated; the copy is user generated and editable.

### Refresh recency evidence
Action: `Actions` -> `Refresh list` on an Active list.
Expected: `List refresh in progress` / `Cancel refresh` available during refresh; toast `<strong>Refresh complete</strong> with <strong>{count}</strong> records: ` + `View updated list`; `Last updated` changes; count reflects the new snapshot (Karate `refresh.feature` add/remove record scenario).

### Landing-page search, filter, sort (C1434637, C1434639, C1434659, C1434660, C1538701)
Action: Type part of a list name in `Search lists` (any case) and press Enter / `Search`; add a `Record types` filter; click the `List name` header.
Expected: matching lists only (name or description, case-insensitive partial match); `<n> records found`; sorted ascending then descending on second click; state persists after opening a list, creating/duplicating a list, reloading, switching apps.

---

## ECS / Multi-Tenant Notes

- Lists app access and data are evaluated per active affiliation: the user needs Lists capabilities and content permissions in each tenant (Confluence 599916623; Cypress `consortia/lists/no-lists-permission-*.cy.js`, `entity-types-permissions-member-tenant.cy.js`).
- Cross-tenant queries are only possible from the **Central** tenant for record types with cross-tenant support: Instances, Holdings, Items (Ramsons), Instances with MARC bibliographic (MODFQMMGR-1106, MODLISTS-250); a member tenant cannot query another member tenant. Data from a tenant is returned only if the user has an affiliation and permissions there.
- Lists with a cross-tenant query must be **Private**: the UI forces `Private` and shows [en.json `create-list.cross-tenant-tip`]; the API rejects shared ones with 400 `list.is.cross-tenant.and.not.private` (`ListValidationService.assertListIsNotCrossTenantAndShared`; UILISTS-175, UILISTS-198, MODLISTS-152). Canned `Missing items` is private and not shown on Central (MODLISTS-198).
- Indicator [en.json `cross-tenant-message`] `This list may contain records from multiple tenants.` on the details page and landing page (UILISTS-173, UILISTS-185; Cypress `cross-tenant-indicator-central-tenant.cy.js`).
- `Instance — Affiliation name` / `Instance — Shared` fields exist only in ECS for Instances/Holdings/Items (MODFQMMGR-396, MODFQMMGR-405; Cypress `verify-affiliation-name-*.cy.js`).
- Authority: Central queries return shared and local authorities (expected, MODFQMMGR-1195 closed as expected); a member tenant sees shared authorities (MODFQMMGR-1113).
- Member tenant ET availability depends on the ET capabilities assigned (C1415950, C1415952; MODFQMMGR-377).

---

## Existing Automated Coverage

- Karate mod-lists (`folio-integration-tests/mod-lists/src/main/resources/athena/mod-lists/features/`): `create.feature`, `get-lists.feature` (ids, `updatedAsOf`, `private`, `active`), `update.feature`, `delete.feature` (soft delete), `refresh.feature` (already refreshing, inactive, cancel, count change), `export.feature` (selected/all columns per ET, MARC bib tag fields, export during refresh), `get-list-contents.feature`, `versions.feature`, `access-control.feature` (private list), `statistical-codes.feature`, `agreement-lines-entity-type.feature`, `agreements-invoices-orders-entity-type.feature`, `consortia/*` (ECS export). Runners: `ModListsTest.java`, `ModListsCriticalPathApiTest.java`.
- Karate mod-fqm-manager (`…/mod-fqm-manager/features/`): `query/operators-basic.feature`, `query/operators-array.feature`, `query/marc-fields.feature`, `query/per-entity-type.feature`, `query/special-field-tests.feature`, `query/validation.feature`, `custom-entity-types.feature`, `migration.feature`, `entity-types.feature`, `consortia/cross_tenant_*_query.feature`, `member_*_query.feature`. edge-fqm: `edge-entity-types.feature`, `edge-query.feature`.
- Not covered by Karate (candidates): `GET /lists` `search`/`canned`/`createdBy`/`updatedBy`/`sortBy`, `GET /lists/related-users`, optimistic lock 409, canned update/delete 400, cross-tenant private enforcement 400, max list size.
- Cypress (`stripes-testing/cypress/e2e/lists/`, 282 specs incl. consortia): `create-list/`, `edit-list/`, `duplicate-list/`, `delete-list/`, `Export-list/`, `refresh-list/`, `lists-details-page/`, `permissions/`, `query-builder/` (per ET folders, operators, MARC under `srs/` and `authority/`), landing-page specs (`verify-search-*`, `verify-sorting-in-lists-table.cy.js`, `created-by-filter.cy.js`, `updated-by-filter.cy.js`, `search-filter-sort-persistence.cy.js`); ECS in `cypress/e2e/consortia/lists/`. Fragments: `cypress/support/fragments/lists/lists.js`, API helpers `cypress/support/api/lists.js`.

---

## Key Business Rules for Test Cases

1. List record sets are snapshots; they change only on refresh or on saving a new query, while values of records already in the set are read at view time. (cases; docs `Lists/_index.md`)
2. Inactive lists cannot be refreshed or exported; refresh returns `refresh-list.is.inactive`, export `export-list.is.inactive`; refresh and export actions are disabled in the UI. (`ListValidationService.java`; C411840, C411811; Karate `refresh.feature`)
3. Duplicate list prepopulates name `<name> - copy`, description, status, visibility and query, and saving creates an independent list. (`useCopyListState.ts`; Cypress `duplicate-list/*`)
4. Refresh updates `Last updated` and the record count; while it runs `List refresh in progress` and `Cancel refresh` are shown. (Karate `refresh.feature` add/remove record scenario; en.json `refresh.status.in-progress`)
5. Export uses the latest successful refresh, not live data. (mod-lists `ListIdsProvider.java` reads contents by `getSuccessRefresh().getId()`)
6. `Actions` options depend on the capability set: View has none, Edit adds Refresh/Edit/Duplicate, Delete adds Delete, Export adds both export actions, Admin has all. (Cypress `permissions/*.cy.js`)
7. Lists app access needs a Lists capability set; ET permissions alone never unlock the app. (C476848)
8. The Query Builder must keep queries editable across edit/duplicate flows; the user-friendly query never shows `undefined` and values show labels, not codes. (UIPQB-242, UIPQB-260; C1259783)
9. Some array fields are not queryable but can be shown as columns and exported; check per ET in docs. (C451499; docs)
10. Concurrent edits are guarded by `version`: the second save fails with `update-optimistic.lock.exception` (409). (C411739; `OptimisticLockException.java`)
11. ET permissions are re-checked on every read/refresh/export/delete; removing one after creation blocks the list. (C476851; `ListValidationService.java`)
12. Value is kept when switching among `equals`/`not equal to`/`starts with`/`contains`, and `in`↔`not in`; switching to or from `is null/empty` resets it. `contains`/`starts with` are case-insensitive. (UIPQB-250, UIPQB-180, MODFQMMGR-994; C651429, C651430)
13. `not equal to` and `not in` return records with null/empty values for string, UUID, date, integer, boolean and array fields. (MODFQMMGR-753/754; C740199, C740217/C740218/C740220)
14. `Authority` is a supported record type with its own field vocabulary. UUID fields show the UUID (not a name) after save and use only `equals`, `in`, `not in`, `is null/empty`. (C1322605; UIPQB-295, MODFQMMGR-1222)
15. New record types since 2026-06: Agreements + Lines, Agreements-Invoices-Orders (MODFQMMGR-1133/1134), Payment terms on PO Line (MODLISTS-248), Request aggregates (MODFQMMGR-1231); custom ETs can join other ETs (MODFQMMGR-896, C1464107).
16. A user holding the list's ET permissions can export it without permission errors. (MODLISTS-276)
17. Canned (System) lists cannot be edited or deleted (API 400 `list.is.canned`; UI `Edit list` disabled) but can be refreshed, exported and duplicated. (`ListValidationService.java`; C411731, C411769, C411810)
18. Delete is blocked while the list is refreshing or exporting; delete is soft (`isDeleted`), and versions/contents are no longer returned. (`ListValidationService.java`; C411770, C411771; Karate `delete.feature`, `versions.feature`)
19. Edit (PUT) is blocked while the list is refreshing or exporting, for every user. (`ListValidationService.validateUpdate`; C411737, C411738, C411766, C411767)
20. Refresh is blocked while another refresh or an export is running and when the list has no query. (`ListValidationService.validateRefresh`; Karate `refresh.feature`)
21. The same user cannot start a second export of a list while theirs is running; a different user can. (`assertUserNotExporting`)
22. Private lists are accessible only to the current owner (`updatedBy`, else `createdBy`); others get 401 `list.is.private`. (`assertSharedOrOwnedByUser`; Karate `access-control.feature`; C411710)
23. Setting a list Inactive deletes its contents and clears the last successful refresh; reactivating requires a manual refresh. (`ListService.java`; en.json `warning.inactive-status`, `callout.list.active`)
24. Each create/update stores a historic version readable via `GET /lists/{id}/versions`; versions follow the list's access control. (Karate `versions.feature`)
25. `GET /lists` returns only lists of permitted ETs; the `Record type` dropdown shows only permitted ETs. (MODLISTS-112; MODFQMMGR-658)
26. Landing-page search matches list name and description, case-insensitive, partial match, and combines with filters and sorting. (UILISTS-252, MODLISTS-68; C1434637, C1434639, C1434660)
27. `Source` filter: `System` returns canned lists only, `User generated` excludes them, none selected returns both. (UILISTS-263, MODLISTS-243)
28. `Created by` / `Updated by` are multi-select dropdowns of users who created/updated lists, sorted alphabetically, and do not need user-view permissions; deleted users are not offered and names are not synced after user edits. (UILISTS-281, MODLISTS-277)
29. Sort by `List name`, `Last updated`, `Records`; lists without a successful refresh sort first ascending and last descending by `Records`. (UILISTS-254, MODLISTS-68 comments)
30. Search term, filters and sort persist across view/add/edit/duplicate/delete, reload and app navigation. (UILISTS-279; C1538701)
31. After saving a changed query, the detail page shows `Query in progress` until the new query finishes and never shows the old count. (UILISTS-276; C1574527)
32. Export is available only from the details page, not from Edit; `Export selected columns (CSV)` contains only shown columns, ID columns only when selected. (UILISTS-235, MODLISTS-232)
33. CSV headers use UI labels (Trillium+) and duplicate value labels get the code appended (`German [de]`). (Confluence 2073591809; MODLISTS-249)
34. A query over `maxListSize` (default 1,250,000) shows `modal.banner.limit` in the Query Builder and the refresh fails with `refresh-max.list.size.exceeded`. (MODFQMMGR-723; mod-lists `README.md`)
35. Deleted records stay in the list until refresh and show `Deleted` with `—` cells and a tooltip; for composite ETs only the missing source's columns are empty. (UIPQB-72; C957375)
36. Legacy array operators (`contains any/all`, `not contains any/all`) no longer exist; saved lists were migrated to `in`/`not in`/`equals`/`not equal to`. (MODFQMMGR-889/898/899/900/901)
37. `in` on free-text strings is offered only for barcodes and HRIDs, and `not in` is not offered for them. (MODFQMMGR-1081)
38. Lists with cross-tenant queries (ECS Central; Instances, Holdings, Items, Instances with MARC bibliographic) are always Private and show the multi-tenant indicator. (UILISTS-175, MODLISTS-152, MODLISTS-250)
39. A custom ET used by a list cannot be deleted (409); a list whose ET is gone is hidden from the landing page and its URL shows `This list's entity type is no longer available`. (MODFQMMGR-959, UILISTS-239)
40. MARC field conditions require a 3-digit tag; indicators accept a-z, 0-9 or `\`; subfield a-z or 0-9; LDR/006/007/008 support whole/position/range. (en.json UIPQB `marc.validation.*`; MODFQMMGR-1190/1224)
41. The Query Builder results table shows values exactly as stored (no space trimming). (UIPQB-303; C1574533)
42. Date fields (`Birth date`, `Enrollment date`, `Expiration date`) are queried, displayed and exported in the tenant time zone, not UTC. (UIPQB-304; C1632042)
43. `User — Address` sub-fields are queryable: `Country` and `Type` use dropdown values (`equals`, `not equal to`, `in`, `not in`, `is null/empty`); `Line 1`/`Line 2`/`Postal code`/`Region`/city use free text (`equals`, `not equal to`, `contains`, `starts with`, `is null/empty`); `Primary address` is a True/False dropdown (`equals`, `is null/empty`). (MODFQMMGR-1043; C1045968)
44. Keyboard/accessibility: `New` is focused exactly once in the Tab order; `View updated list` is reachable by Tab and activated by Enter and Space; the Build query modal has no hidden or zero-size Tab stops (also when reopened via `Edit query`) and the title is not clipped under WCAG 1.4.12 text spacing. (UILISTS-266, UILISTS-267, UIPQB-289, UIPQB-290, STCOM-1552; C1505043)

---

## Authoring style (measured 2026-07-23)

Lists (FQM): **`Other` ~65%** / Func ~35%, median ~6 steps, `User Journey` ~0%. Cases build/test/refresh/export a query-based list or check a query-builder operator/field behavior. `Other` for list management (create/duplicate/refresh/export, capability gating); `Functional` for query-semantics behavior (operator families, null handling, value display). Preconditions carry the record data the query targets and the Lists capability sets. Query-builder cases assert exact match counts before/after.

---

## Known Gaps / Items to Verify

- [ ] TestRail (project 14, suite 21) only partially read: on 2026-11-24 the MCP was offline; on 2026-11-25 only C1045968, C1632042, C1505043 were read in full. Sections 29149 (1 case), 29154 (15 cases: C411731–C411741, C411765–C411767, C1574527) and 34769 (8 cases) were listed by title; other Lists subsections (create, refresh, export, delete, permissions, query builder per ET; IDs not listed because the suite section tree is paginated) and all other case bodies remain unread. Re-run the TestRail pass.
- [ ] Detail page after ET permission removal: C476851 (older) expects the generic `You don't have permission to view this app/record`; current code has [en.json `error-component.read-entity.type.restricted`] `You do not have permission to view this list` (UILISTS-128). Verify which appears.
- [ ] Export headers and ID columns: C451513 allows snake_case headers (`loans.status_name`) and unselected ID columns; Trillium notes (Confluence 2073591809) and MODLISTS-232 say headers use UI labels and ID columns appear only when selected. Update C451513 or verify in env.
- [ ] `User — Address`: MODFQMMGR-1043 (Trillium) made it queryable and C1045968 (updated 2026) confirms it; C451499 and docs.folio.org (`Lists/_index.md`, Users table: "NO") still say not queryable — update C451499/docs.
- [ ] Application name for Lists capability sets: docs.folio.org says `app-fqm`; Confluence 572948576 says `app-lists` for Trillium roles. Check the tenant.
- [ ] `Reset all` and sort: UILISTS-254 refinement says Reset all does not reset the sort, its scenario says results return to the default sort. Verify (Cypress `verify-reset-and-clear-search-box.cy.js`, C1434638).
- [ ] Default landing-page filter appears to be `Status: Active` (URL `filters=status.Active` in UILISTS-239 comments); confirm before asserting.
- [ ] `Updated by` semantics: UILISTS-253 asks whether a refresh counts as an update ("Last refreshed(?)"); not confirmed by a source.
- [ ] docs.folio.org "Duplicate a list" says the toast is `List Original List Name - copy was saved`; en.json has `List <strong>{listName}</strong> saved.` — assert the en.json text.
- [ ] Whether `Delete list` is hidden or disabled for canned lists in the UI is not shown by a source (Cypress only asserts `Edit list` disabled).
- [ ] Capability naming variants (`Module Lists + Manage`, `Module Lists Refresh + view`, casing) appear in older case preconditions (e.g. C1505043); use the table above. `Module Lists Enabled - View` and `Module Lists - Delete` are confirmed only by docs.folio.org, not by the Cypress dictionary. Content-permission capability set names per entity type are not confirmed.

---

## Change log

- 2026-11-25 remediation: source tags added (rules 4, 5, 8), capability table gets confirming sources, rules 45-47 added from C1045968/C1632042/C1505043, 3 duplicate rules merged into rules 8, 12, 15 and 4 new/extended rules added (45-47 equivalents: tenant time zone dates, User Address sub-fields, accessibility; now rules 42-44), release header form fixed. Rules 44.
- 2026-11-24 full refresh: re-verified all UI strings against en.json (corrected the refresh-complete toast — `Refresh complete` is bold); added API surface, error codes, export/refresh/cancel and private-list errors, ECS cross-tenant rules, custom ET rules, search/filter/sort rules, Karate/Cypress coverage; corrected capability-set table (Delete = `Module Lists - Delete`, View = `Module Lists Enabled - View`); widened value-retention rule per UIPQB-250; moved export header/ID-column and User Address conflicts to Known Gaps. Rules 17 -> 44.
- 2026-11-07 delta (merged): UILISTS-252/253/254/263/279/280/281/277; UIPQB-278/287/277, MODFQMMGR-1141/1143/1188/1190/1191/1211; MODFQMMGR-1133/1134, MODLISTS-248, MODFQMMGR-1113/1231; UIPQB-295/297/298/299, MODFQMMGR-1208/1215, MODLISTS-276, MODFQMMGR-1150. Reference cases created 2026-06-17…2026-11-07 include C1599856–C1599869 (MARC LDR/control fields), C1574533, C1574527, C1559243/C1559244, C1538701/C1538703, C1505019/C1504486 (localization), C1464107, C1434636–C1434657 (search & filter), C1415950/C1415952 (ECS member ETs), C1405043–C1405046 (linked bib counts).
- 2026-07-21/23 audits: C476848, C476851, C451499, C451514, C651429, C651430, C740199, C831960, C1259783, C1312673, C411739, C451513, C1322605.
