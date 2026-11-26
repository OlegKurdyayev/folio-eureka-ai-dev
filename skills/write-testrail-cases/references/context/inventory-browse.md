---
area: inventory-browse
generated_on: 2026-11-25
generator_version: manual (build-app-context method, refresh)
jira_status: ok
---

# FOLIO Inventory Browse — Business Logic Context

> Built: 2026-11-07 · Refreshed: 2026-11-25 · Release: Umbrellaleaf R2 2026 (master: ui-inventory 14.1.0 IN PROGRESS, stripes-inventory-components 2.1.0, mod-search master; last released: Trillium R1 2026 = UIIN 14.0.x / MSEARCH 6.0.x) · Builder: build-app-context method, manual
> Sources: TestRail suite 21 sections 17147 "Call Number Browse" (55 cases) + 55325 "Browse configuration checks" (18), 17148 "Subject Browse" (63), 17888 "Contributors Browse" (45), 35111 "Instance classification browse" (52), Settings 55320 "Call number browse" (5) and 35321 "Classification browse" (7) — titles for all, full steps for C347902, C375064, C451641, C627455, C805650, C805746, C374171, C451640, C451643, C353647; GitHub folio-org/ui-inventory (`translations/ui-inventory/en.json`, `src/filterConfig.js`, `src/components/BrowseResultsList/*`, `src/components/InstanceFiltersBrowse/InstanceFiltersBrowse.js`, `src/hooks/useInventoryBrowse/constants.js`, `src/hooks/useBrowseValidation/useBrowseValidation.js`, `src/views/BrowseInventory/BrowseInventory.js`, `src/settings/*BrowseSettings/*`, `package.json`); folio-org/stripes-inventory-components (`lib/constants/queryIndexes.js`, facet components, `translations/stripes-inventory-components/en.json`); folio-org/mod-search (`descriptors/ModuleDescriptor-template.json`, `src/main/resources/swagger.api/`); folio-integration-tests `mod-search/.../browse/*.feature`; stripes-testing `cypress/e2e/inventory/*browse*`; Jira UIIN, MSEARCH Done stories/bugs for Ramsons (UIIN 12 / MSEARCH 4), Sunflower (13 / 5), Trillium (14 / 6) and master (UIIN 14.1.0), ~85 issues with details; docs.folio.org Inventory "Browsing for a record".
> Related contexts: `inventory.md` (records, search, move), `marc-authority.md` (authority browse — a different app), `marc-bib-quickmarc.md` (linking that drives the authorized indicator).

## What is Browse

`Inventory` → `Search | Browse` toggle → `Browse`. The left pane title is `Browse inventory` (en.json `title.browseCall`; subtitle `Enter search criteria to start browsing`, `title.subTitle.browseCall`); the empty results pane shows `Browse for results entering a query or choosing a filter.` (`notLoadedMessage.browseCall`). Page title: `Inventory - {query} - Browse` (`documentTitle.browse`; C423413). Browse lists index headings around the query (anchor row in bold, 5 preceding rows, 100 rows per page — C347902; `PRECEDING_RECORDS_COUNT = 5` in `useInventoryBrowse/constants.js`), not instances; clicking a heading runs an instance search. On landing, `Search` and `Reset all` are disabled and the `Actions` menu is hidden (C347902).

## Browse options (`Select a browse option`, en.json `browse.searchableIndexesPlaceholder`)

Order from `src/filterConfig.js` (`browseInstanceIndexes`) and C347902:

| Group heading (en.json) | Option (en.json key) | API path (`PATH_MAP`) | Facets (`InstanceFiltersBrowse.js`) |
|---|---|---|---|
| `Call numbers (item)` (`browse.callNumbers`) | `Call numbers (all)` (`browse.callNumbersAll`) | `browse/call-numbers/all/instances` | `Shared` (member tenant only), `Held by` (ECS only), `Effective location (item)` |
| | `Dewey Decimal classification` (`browse.dewey`) | `.../dewey/instances` | same |
| | `Library of Congress classification` (`browse.libOfCongress`) | `.../lc/instances` | same |
| | `National Library of Medicine classification` (`browse.natLibOfMed`) | `.../nlm/instances` | same |
| | `Other scheme` (`browse.other`) | `.../other/instances` | same |
| | `Superintendent of Documents classification` (`browse.superintendent`) | `.../sudoc/instances` | same |
| `Classification (instance)` (`browse.classification`) | `Classification (all)` (`browse.classification.all`) | `browse/classification-numbers/all/instances` | `Shared` (member tenant only); no Held by, no location (UIIN-2813) |
| | `Dewey Decimal classification` (`browse.classification.dewey`) | `.../dewey/instances` | same |
| | `Library of Congress classification` (`browse.classification.lc`) | `.../lc/instances` | same |
| — | `Contributors` (`browse.contributors`) | `browse/contributors/instances` | `Shared` (member only), `Name type` |
| — | `Subjects` (`browse.subjects`) | `browse/subjects/instances` | `Shared` (member only), `Subject source`, `Subject type` (UIIN-2961) |

- There is no `Local` call number browse option: `browse.local` is still in en.json, but the option is not in `filterConfig.js` (`PATH_MAP` comment: "not supported, option disabled") and mod-search `browseOptionType` has no `local` value. Local-type call numbers are browsed through `Call numbers (all)` or through options whose Settings include the local type.
- In a non-ECS tenant, call number options show only `Effective location (item)` (C347902 step 5). Facet labels come from stripes-inventory-components en.json: `Shared` (`facet.shared`), `Held by` (`facet.tenantId`), `Effective location (item)` (`facet.effectiveLocation`), `Name type` (`facet.nameType`). `Subject source` / `Subject type` come from ui-inventory `subjectSource` / `subjectType`.
- `Held by` for Contributors/Subjects is commented out in code ("until back-end requirements and implementation are done"; `InstanceFiltersBrowse.js`).
- Contributors browse is offered on the Instance tab only, not on Holdings/Item (C353642). Switching browse options keeps the query and does not submit the form (C387486, C358148). Switching to an option from a different group (call numbers ↔ classification ↔ contributors ↔ subjects) clears the selected facets (`BrowseInventory.js` `onChangeSearchIndex`; UIIN-3499).

## Result list

Columns from `VISIBLE_COLUMNS_MAP` (`src/components/BrowseResultsList/constants.js`):

| Option | Columns (en.json key) |
|---|---|
| Call numbers (all six options) | `Call number` (`instances.columns.callNumber`), `Title` (`instances.columns.title`), `Number of titles` (`instances.columns.numberOfTitles`) |
| Classification (three options) | `Classification` (`instances.columns.classificationNumber`), `Title`, `Contributors` (`instances.columns.contributors`), `Number of titles` |
| Contributors | `Contributor` (`instances.columns.contributor`), `Type` (`instances.columns.contributorType` — the name type), `Relator term` (`instances.columns.relatorTerm`), `Number of titles` |
| Subjects | `Subject` (`subject`), `Subject source` (`subjectSource`), `Subject type` (`subjectType`), `Number of titles` |

- Call number cell = prefix + call number + suffix joined by spaces (`getFullCallNumber` in `utils.js`); `fullCallNumber` in the index also covers item enumeration data (C478253, C478256).
- `Title` is shown only when the heading belongs to one instance; blank when it belongs to more than one (MSEARCH-948, UIIN-3367). `Contributors` lists all instance contributors separated by ` ; `, blank when the instance has none or the classification is on several instances (UIIN-3368; `getBrowseResultsFormatter.js`; C805650, C805746).
- No match: a placeholder row `{query} would be here` (`browseCallNumbers.missedMatch`; C347916, C350392). The anchor row (exact match) is bold (C347902).
- The heading is a hyperlink (C374701…C374704). Rows with `totalRecords` 0, contributors without a name type and subjects without records are not clickable (`isRowPreventsClick` in `utils.js`).
- `Number of titles` = 1 opens the instance in the third pane; more than one opens the Search results (C411505…C411507).
- Hit count is not shown on browse lists (C358531, C356786).
- `Previous` / `Next` pagination; disabled on the first/last page; leaving the first/last page works (C353643, C468259…C468261; UIIN-2992, UIIN-3508).
- Browse results and filters persist when switching to Search and back, after browser Back and after page reload (C380402…C380405, C357033, C347902 step 8).
- `Reset all` returns focus to the search box and does not affect the Search pane (C422098, C422099, C398011; `BrowseInventory.js` `onReset`).
- Authorized indicator: for contributors (100/110/111/700/710/711) and subjects (600/610/611/630/650/651/655) controlled by a MARC authority, the MARC authorities app icon is shown before the heading with tooltip `Linked to MARC authority` (en.json `linkedToMarcAuthority`); clicking the icon opens the authority record in a new tab (`getBrowseResultsFormatter.js`; C375152…C375254). Linked and unlinked headings with the same text are separate rows (C375163).
- Contributors with the same name and name type but different authority IDs are separate rows; same authority ID — one row (C359595, C359596, C388531). Same name, same instance, same name type, different relator terms — counted once (C353660); different name types — two rows (C353999).
- Subjects are split by heading + source + type; `Number of titles` counts instances with that exact combination (UIIN-2960, UIIN-3101; C584543…C584546).

## Browse → Search redirect (`src/components/BrowseResultsList/utils.js` `getSearchParams`)

| Option | Search index and query |
|---|---|
| Call numbers | Query search `itemFullCallNumbers="<full call number>"`, plus `item.effectiveCallNumberComponents.typeId==…` for each type selected in Settings for that option (UIIN-3234: `itemFullCallNumbers` replaced `itemNormalizedCallNumbers`, which right-truncated; shelving order is no longer used — UIIN-2906) |
| Classification | Query search `classificationId=="<id>"`, plus `classifications.classificationTypeId==…` for the configured types (UIIN-3516, MSEARCH-1093, master 14.1.0) |
| Contributors | `Contributor` index with the name + filter on the row's name type (C357548, C358966) |
| Subjects | `Subject` index with the heading + filters on authority ID, subject source and subject type of the row (UIIN-3101) |

Browse facets `Shared` and `Held by` are carried to Search as `shared.*` / `tenantId.*` filters. Double quotes and backslashes in the heading are escaped (`escapeCqlValue`; UIIN-3537, UIIN-3530).

## Sorting and matching rules

- Call number browse is case-insensitive (MSEARCH-859; C594363, C594365) and ignores spaces in SuDoc (MSEARCH-1005) and in Dewey, Other scheme and All sorting (MSEARCH-1262, examples: `331.198` = `331. 198`; `B JORDAN DVD` = `BJORDANDVD`; `QS 11 .GA1 E53 2005` = `QS11.GA1E53 2005`).
- Classification browse is case-insensitive; `Classification (all)` sorts alphabetically, Dewey and LC use their own algorithms (C468241…C468258). Shelving algorithms in mod-search: `lc`, `dewey`, `nlm`, `sudoc`, `default` (`shelvingOrderAlgorithmType.yaml`).
- Double quotes, diacritics, backslash and a single-letter query are supported (UIIN-3537, UIIN-3530, MSEARCH-1094; C347915, C360948, C466294, C466296, C414511, C869994, C916258).
- `*` in a Contributors query shows the error toast `Error returning results. Please retry or revise your search.` (`browseContributors.results.error`; `useBrowseValidation.js`; C358151). Other options have no UI validation for `*`; browse never supported it (MSEARCH-1022, Won't Do).
- Special characters that the LC algorithm strips (e.g. `+`) give an empty shelving order and sort to the end in the LC option; such call numbers remain browsable in `Call numbers (all)` (MSEARCH-1150, Won't Do).
- Same call number with different copy numbers (enumeration) — separate rows (C651491, C651492). Same call number in two holdings with different locations of one instance — one row, `Number of titles` 1, title shown (MSEARCH-992; C692245).
- Call numbers with an invalid format for their type are still found (C451466…C451470). Trailing spaces cannot be saved through the UI (UIIN-2889; C543802…C543806); Data import can still create them.
- Classification with the same value but different types: one row per type in typed options, adjacent rows in `Classification (all)` (UIIN-2907; C468158…C468160).

## Settings

| Page (`Settings > Inventory`) | Rows | Column | Toast (en.json) |
|---|---|---|---|
| `Call number browse` (`callNumberBrowse`), above `Call number types` in "Holdings, Items" | `Call numbers (all)`, `Dewey Decimal classification`, `Library of Congress classification`, `National Library of Medicine classification`, `Other scheme`, `Superintendent of Documents classification` (C627455; `settings.instanceCallNumber.*`) | `Call number types` multi-select, empty by default | `The call number browse type {term} was successfully updated` (`settings.instanceCallNumber.termUpdated`) |
| `Classification browse` (`classificationBrowse`), above `Classification identifier types` | `Classification (all)`, `Dewey Decimal classification`, `Library of Congress classification` (C451641) | `Classification identifier types` multi-select, empty by default | `The Classification browse type {term} was successfully updated` (`settings.instanceClassification.termUpdated`) |

- Info popovers: `Please note that if no call number types are selected for a browse option, this option will display all call number types.` and `Please note that if no classification identifier types are selected for a browse option, this option will display all classification identifier types.` (en.json `*.identifierTypesPopover`; C627455, C451641).
- Rows cannot be created or deleted, only edited (pencil); Name is read-only (`CallNumberBrowseSettings.js`: `canCreate={false}`, delete suppressed).
- Only selected types appear in that option's results; empty means all (C627499…C627509, C468147…C468157).
- Deleting a call number type or a local classification identifier type removes its ID from the browse config (MSEARCH-863; C449381).
- More than 13 classification types can be saved for one option (MSEARCH-890).
- ECS: the config is read and saved in the central tenant (`tenant={centralTenantId}`); the page is shown only when the user has the settings permission in both the current and the central tenant (`CallNumberBrowseSettings.js`; UIIN-3038).

## ECS notes

- `Shared` facet is shown only in a member tenant (`SharedFacet.js` `checkIfUserInMemberTenant`); central tenant has no facets for classification browse (UIIN-2813).
- `Held by` (call number options only): in a member tenant, selecting a call number option preselects the current tenant; central tenant — nothing preselected; `Reset all` clears it (UIIN-3457, `BrowseInventory.js`). The facet was removed in Sunflower CSP (UIIN-3301) and restored (UIIN-3414).
- `Effective location (item)` options show the tenant name (UISINVCOMP-72, MSEARCH-1044). `Shared` and `Effective location (item)` options update each other (MSEARCH-1012).
- Promoted-to-shared instance: its call number counts as shared (MSEARCH-1168); headings are not double-counted as shared and local (MSEARCH-941).
- After `Update ownership` of holdings/items the call number stays browsable (MSEARCH-1085).
- Known accepted limitations (Won't Do): `Number of titles` counts instances from all tenants regardless of `Held by` (MSEARCH-1046); facet counts with the same local call number in two member tenants (MSEARCH-994); locations of other tenants shown with `Shared` = No (MSEARCH-1018); no results when `Shared` + `Held by` + `Effective location (item)` are all applied (MSEARCH-1008). Do not write these as expected-pass checks.

## API surface (mod-search `ModuleDescriptor-template.json`)

| Method / path | permissionsRequired |
|---|---|
| GET `/browse/call-numbers/{browseOptionId}/instances` (`all`, `lc`, `dewey`, `nlm`, `sudoc`, `other`) | `browse.call-numbers.instances.collection.get` |
| GET `/browse/classification-numbers/{browseOptionId}/instances` (`all`, `lc`, `dewey`) | `browse.classification-numbers.instances.collection.get` |
| GET `/browse/contributors/instances` | `browse.contributors.instances.collection.get` |
| GET `/browse/subjects/instances` | `browse.subjects.instances.collection.get` |
| GET `/browse/config/{browseType}` (`instance-call-number`, `instance-classification`) | `browse.config.collection.get` |
| PUT `/browse/config/{browseType}/{browseConfigId}` | `browse.config.item.put` |
| GET `/search/{recordType}/facets` (`call-numbers`, `classifications`, `contributors`, `subjects`) | `search.facets.collection.get` |

- Query params (mod-search `src/main/resources/swagger.api/`): `query` (CQL anchor on `fullCallNumber` / `number` / `name` / `value`, e.g. `fullCallNumber < "X" or fullCallNumber >= "X"`), `limit` (0–100, default 100), `precedingRecordsCount` (1–100), `highlightMatch` (default true). Call number filters: `instances.tenantId`, `instances.shared`, `instances.locationId`, `callNumberTypeId`.
- Response items (mod-search `swagger.api/` schemas): call number — `id`, `fullCallNumber`, `callNumber`, `callNumberPrefix`, `callNumberSuffix`, `callNumberTypeId`, `instanceTitle`, `isAnchor`, `totalRecords`; classification — `id`, `classificationNumber`, `classificationTypeId`, `instanceTitle`, `instanceContributors`, `isAnchor`, `totalRecords`; contributor — `id`, `name`, `contributorTypeId[]`, `contributorNameTypeId`, `authorityId`, `isAnchor`, `totalRecords`; subject — `id`, `value`, `authorityId`, `sourceId`, `typeId`, `isAnchor`, `totalRecords`. Browse config: `id`, `shelvingAlgorithm`, `typeIds[]`.
- New headings reach browse through a background job, not instantly (MSEARCH-1168: steps wait at least 1 minute after sharing). Classification `instanceTitle`/`instanceContributors` need a reindex of `instance` + `classification` after upgrade (MSEARCH-1045).

## Permissions

| Permission (ui-inventory package.json) | Display name | Key sub-permissions |
|---|---|---|
| `ui-inventory.call-number-browse.view` | `Call number browse: View permissions` | `ui-inventory.instance.view`, `browse.call-numbers.instances.collection.get`, `browse.authorities.collection.get` |
| `ui-inventory.subjects.view` | `Subject browse: View permissions` | `ui-inventory.instance.view`, `browse.subjects.instances.collection.get`, `browse.authorities.collection.get` |
| `ui-inventory.settings.call-number-browse` | `Settings (Inventory): Configure call number browse` | `browse.config.collection.get`, `browse.config.item.put`, `inventory-storage.call-number-types.collection.get` |
| `ui-inventory.settings.classification-browse` | `Settings (Inventory): Configure classification browse` | `browse.config.collection.get`, `browse.config.item.put`, `inventory-storage.classification-types.collection.get` |
| `ui-inventory.instance.view` | `Inventory: View instances, holdings, and items` | includes all four browse endpoints and `browse.config.collection.get` (MSEARCH-965) |

Eureka capabilities: `Data - UI-Inventory - Manage` for `Inventory: All permissions` (stripes-testing `capabilitySets.js` `uiInventory`; C805650, C805746); `Data - UI-Inventory Instance - View` (`uiInventoryInstanceView`) for instance view (`capabilitySets.js`); `Settings - UI-Inventory Settings Call-Number-Types - View` (`capabilitySets.js` `uiInventorySettingsCallNumberTypesView`; C627455); `Settings - UI-Inventory Settings Call-Number-Browse - View` for `Settings (Inventory): Configure call number browse` (confirmed only by the C627455 precondition table, spelled there in lowercase `settings - ... - view`; not in the Cypress dictionary). Capability set names for `Call number browse: View permissions`, `Subject browse: View permissions` and `Settings (Inventory): Configure classification browse`: (capability set name not confirmed — verify) (Okapi permission names only in `permissions.js` `uiCallNumberBrowse`, `uiSubjectBrowse`, `uiInventorySettingsConfigureClassificationBrowse`; cases C374171, C451640, C451643 list no capability). Users with only a browse permission cannot edit records (C375064); without `Subject browse: View permissions` the `Browse subjects` option is not offered even with `Inventory: View instances, holdings, and items` (C374171, UIIN-2034 — the case text predates the combined browse option list, re-check wording); without `Settings (Inventory): Configure classification browse` the `Classification browse` section is not displayed in Settings > Inventory (C451640, UIIN-2775).

## Existing automated coverage

- Karate (folio-integration-tests): `mod-search/src/test/resources/promin/mod-search/browse/call-number-browse.feature`, `contributor-browse.feature`, `subject-browse.feature` (around / around-including / forward / backward, `precedingRecordsCount`, `highlightMatch`), `filters/facet-search.feature`. No Karate feature for classification browse or `/browse/config`.
- Cypress (stripes-testing): `cypress/e2e/inventory/call-number-browse/` (62 files incl. `sorting/`), `classification-browse/` (12), `contributors-browse/` (32), `subject-browse/` (54), `settings/call-number-browse/`, `settings/classification-browse/`; ECS: `cypress/e2e/consortia/inventory/{call-number,classification,contributor,subject}-browse/`, `consortia/settings/classification-browse/`; permissions: `cypress/e2e/permissions/inventory/browse-*.cy.js`. Fragments: `cypress/support/fragments/inventory/search/browseCallNumber.js`, `browseClassifications.js`, `browseContributors.js`, `browseSubjects.js`.

## Key Business Rules for Test Cases

1. Browse shows headings, not records; assert the heading row, `Number of titles` and the anchor (bold) position, then the Search result after clicking (C347902, C411505…C411507).
2. Each browse option is a separate case in the team corpus (all / Dewey / LC / …) — do not merge options into one case (TestRail sections 17147, 35111, 55325).
3. Settings limit call number and classification browse options to the selected types; empty means all (en.json `*.identifierTypesPopover`; C627504…C627509, C468155…C468157).
4. Case does not change results for call numbers and classifications (MSEARCH-859, C468255…C468258).
5. Spaces are ignored in SuDoc, Dewey, Other scheme and All call number sorting (MSEARCH-1005, MSEARCH-1262; C1405529, C1405530).
6. Double quotes and backslashes are found as exact match and survive the redirect to Search (UIIN-3537, UIIN-3530, MSEARCH-1094; C916258, C869994).
7. Facets and query survive Search ↔ Browse switching, browser Back and page reload (C380402…C380405, C357033, C347902).
8. Switching to an option of another browse group clears facets; within a group facets stay (BrowseInventory.js; UIIN-3499).
9. Classification browse shows `Title` and `Contributors` (` ; `-separated); both blank when the classification is on more than one instance or the instance has no contributors (UIIN-3367, UIIN-3368; C805650, C805746).
10. Call number `Title` is blank when the call number is on more than one instance (MSEARCH-948).
11. Call number redirect searches `itemFullCallNumbers` (exact) limited to the configured types, so Search returns exactly `Number of titles` instances (UIIN-3234; C466214).
12. Classification redirect searches by classification ID and configured types (UIIN-3516, master 14.1.0; C466323, C468158).
13. Subject rows are per heading + source + type; the redirect filters by source, type and authority ID (UIIN-2960, UIIN-3101; C584543…C584546).
14. Contributor redirect also filters by the row's name type (C357548, C358966).
15. `*` is rejected only for Contributors with the `browseContributors.results.error` toast (useBrowseValidation.js; C358151).
16. No `Local` call number browse option exists; test local types through `Call numbers (all)` or Settings (filterConfig.js; C347902; C414983).
17. In a non-ECS tenant call number options show only `Effective location (item)`; classification options show no facets (C347902; InstanceFiltersBrowse.js).
18. ECS member tenant: `Held by` preselects the current tenant on call number options; central tenant preselects nothing (UIIN-3457).
19. ECS `Shared` facet is shown only in member tenants (SharedFacet.js; UIIN-2813).
20. Browse settings in ECS are central-tenant data and need the permission in central and current tenant (CallNumberBrowseSettings.js; UIIN-3038).
21. Browse-settings rows cannot be added or deleted; only `Call number types` / `Classification identifier types` are editable (CallNumberBrowseSettings.js; C627455, C451641).
22. Authorized indicator = MARC authority icon with `Linked to MARC authority` tooltip, opens the authority in a new tab (en.json `linkedToMarcAuthority`; C375152…C375254).
23. Rows with 0 titles are not clickable (utils.js `isRowPreventsClick`).
24. Pagination: `Previous`/`Next` disabled on the first/last page; 100 rows per page with 5 preceding (C347902, C468259…C468261, UIIN-3508).
25. Hit count is not shown on browse lists (C358531, C356786).
26. After record create/update/delete the browse list reflects the change after a short delay (background indexing) — add a wait step (C651502, MSEARCH-1168).
27. Regression scope for browse changes: the Search redirect counts (C466190, C466200, C466214) and the `Effective location (item)` counters (C411749, C651497).
28. Recent bug areas to retest: Classification browse 400 after reindex (MSEARCH-1252), unindexed classifications (MSEARCH-1270), bookmark with collapsed filter pane (UIIN-3603), facet leaking between forms (UIIN-3499).

## Authoring style (measured 2026-11-07, sampled 2026-11-24)

Types `Functional`/`Smoke` (type_id 6/7 sampled), Priority Medium–High. Recent cases (C627455, C805746) use the Okapi/Eureka permission table with `Capabilities/Sets for Eureka env. (type/table - resource - action)`; older ones list Okapi permission names only (C347902, C375064). Preconditions: permissions table, test records with specific call numbers/classifications/subjects (often a MARC file imported with the default job profile), `"Browse" tab is selected in "Search|Browse" toggle`, the browse option selected. Steps are short (2–11), each with an example query value (`ex.: PS153.N5 P443 2025`) and the exact expected row content.

## Known Gaps / Items to Verify

- [ ] Eureka capability names for `Call number browse: View permissions`, `Subject browse: View permissions` and `Settings (Inventory): Configure classification browse` — not found in the Cypress dictionary or in the cases read (C374171, C451640, C451643, C627455) (capability set name not confirmed — verify).
- [ ] Relator term filter for Contributors: C353647 / C353648 / C353658 describe a `Relator term` filter, but `InstanceFiltersBrowse.js` renders only `Shared` and `Name type` for Contributors. Confirm in env before reusing those cases. C353647 (UIIN-2022) also expects a `<Number> records found` header count and a `Contributor` sort, which conflicts with the no-hit-count rule below (C358531, C356786) — treat it as outdated.
- [ ] Contributor `Type` column: en.json key `instances.columns.contributorType` = `Type`, while older cases call it "Name type" — assert `Type`.
- [ ] C350378 / C350379 expect "Browse subjects" on Holdings/Item tabs; current UI has one browse option list regardless of segment — case is likely outdated.
- [ ] C405529 "clicking on Call number executes search by Shelving order" contradicts the `itemFullCallNumbers` redirect (UIIN-3234, UIIN-2906).
- [ ] docs.folio.org still lists `Local` among call number browse types and says call number browse is based on shelving order — conflicts with code (filterConfig.js, UIIN-3234).
- [ ] Open work on master: redirect by call-number / contributor / subject ID (UIIN-3726…3728, MSEARCH-1296/1297); Browse → Search box clearing returns whole catalog (UIIN-3722, In Refinement).
- [ ] Mapping of Umbrellaleaf R2 2026 to ui-inventory version (14.1.0 vs 15.0.0) not confirmed in Jira versions.
- [ ] Hard return inside a call number is not browsable unless included in the query (UIIN-2889 comment) — no case.
- [ ] Authority browse in MARC authority app is out of scope here — use `marc-authority.md`.
