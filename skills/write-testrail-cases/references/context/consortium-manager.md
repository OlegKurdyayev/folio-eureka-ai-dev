# FOLIO Consortium Manager App - Business Logic Context

> Built: 2026-06-18 · Refreshed: 2026-11-25 · Release: Umbrellaleaf R2 2026 (Jira fixVersion "Umbrellaleaf (TBD)", ui-consortia-settings 3.2.0 in progress) · Builder: build-app-context/manual (full refresh, earlier 2026-11-07 delta folded in)

> Sources (2026-11-24 refresh):
> - GitHub `folio-org/ui-consortia-settings` master (v3.2.0 in progress, HEAD 2026-11-19): `translations/ui-consortia-settings/en.json` (primary; `en_US.json` lags by the 7 newest keys), `package.json` (stripes.permissionSets, route `/consortia-settings`), `CHANGELOG.md`, `src/routes/ConsortiumManager/**`, `src/components/ConsortiaControlledVocabulary/**`, `src/constants.js`.
> - GitHub `folio-org/mod-consortia-keycloak` master (Eureka backend): `descriptors/ModuleDescriptor-template.json`, `src/main/resources/swagger.api/**`, `service/BaseSharingService.java`, `domain/dto/SourceValues.java`. (`folio-org/mod-consortia` = legacy Okapi backend, not re-read.)
> - Jira (anonymous REST): UICONSET, MODCON, MODCONSKC Done/Closed stories+bugs resolved >= 2024-06-01 (UICONSET+MODCON 58, MODCONSKC 57; fixVersions hold module versions, so scoped by resolution date) plus open UICONSET bugs; details read for ~45 rule-carrying issues and linked UISAUTHCOM-86.
> - folio-integration-tests `mod-consortia/` (Karate, thunderjet); stripes-testing `cypress/e2e/consortia/consortium-manager/`, `cypress/e2e/eureka/consortia/consortium-manager/`, `cypress/support/fragments/consortium-manager/`, `cypress/support/fragments/settings/authorization-roles/authorizationRoles.js`.
> - docs.folio.org source (`folio-org/docs`): `content/en/docs/Consortia/_index.md`, `content/en/docs/Settings/Settings_consortium-manager/settings_consortium-manager.md`.
> - TestRail (project 14 / suite 21, read-only, 2026-11-25 remediation): section 40125 "Consortium manager (Eureka)" (56 cases, titles listed) and 137898 "Custom links > Consortia" (12 cases, titles listed); case bodies read: C1528153, C1528154, C1528156, C1528157, C1528158, C1528159, C1554377, C1554378, C1554379, C1554380, C722382, C543762, C552447, C523612. All other C-ids below come from the 2026-06-18/07-22 builds and stripes-testing spec titles (bodies not re-read).
> Capability-set names: confirmed only when present in `stripes-testing/cypress/support/dictionary/capabilitySets.js` (or marigoldCapabilities.js) or a TestRail precondition (C-id cited); otherwise marked `(capability set name not confirmed — verify)`.
> Tags: [en.json `key`] = exact string from translations; [C…] = TestRail case; [Cypress …] = stripes-testing fragment/spec; Jira keys and repo paths as given.

---

## What is Consortium Manager

Consortium Manager (`ui-consortia-settings`, route `/consortia-settings`, app title `Consortium manager` [en.json `meta.title`]) is the ECS administration app used from the central tenant to view and manage settings across the member tenants the user is affiliated with: controlled vocabularies (Inventory, Users, Circulation), authorization roles/policies (Eureka), and Data import / Data export logs per member. The same UI module also contributes the **Settings > Consortium manager** area (Network settings: Membership, Network ordering) and the **Switch active affiliation** user-menu action.

Backend: `mod-consortia-keycloak` (Eureka) provides `consortia` 1.1 interface — tenants, user-tenants (affiliations), publish coordinator (`/publications`) and sharing endpoints (`/sharing/settings|roles|policies|instances`). Every write to member tenants goes through the publish coordinator, executed by the `mod-consortia-keycloak` system user (BaseSharingService.java; MODCONSKC-19).

---

## Key Terms

| Term | Definition |
|---|---|
| ECS | Enhanced Consortial Support multi-tenant architecture in FOLIO. |
| Central tenant | Consortium administrative tenant where shared actions are initiated; shown with suffix `(Primary)` in affiliation lists [en.json `affiliation.primary.suffix`]. |
| Member tenant | Affiliated library tenant receiving shared settings and/or local updates. |
| Shared setting | Setting saved with **Share** checked; stored in every consortium tenant with `source = consortium`; Member libraries column shows `All` [en.json `consortiumManager.all`]. |
| Local setting | Setting saved for the selected member(s) only (Share unchecked); one row per tenant. |
| Select members | Header button/modal that scopes which members' data are listed and which members a new local setting is saved to. |
| Affiliation | Tenant a user is associated with (user-tenant record); the active affiliation is the tenant the user currently works in. |
| Publish coordinator (PC) | `POST /consortia/{id}/publications` — fans one request out to several tenants; status `IN_PROGRESS` → `COMPLETE` / `ERROR`. |
| Centrally managed | Key-value on a role/policy detail view; `Yes` when the record was shared (role type `CONSORTIUM`). |
| System user | `System, System user - mod-consortia-keycloak` — shown as "Last updated" by for shared records [Cypress `support/constants/consortia/index.js` CONSORTIA_SYSTEM_USER; UICONSET-231]. |

---

## Record Hierarchy / Architecture

Consortium manager (`src/routes/ConsortiumManager/constants.js` CONSORTIUM_MANAGER_SECTIONS_MAP)
- Header: `Select members` button; subtitle `{amount} members selected`; title `Settings for selected members can be modified at the same time` [en.json `consortiumManager.members.header.*`]
- Management pane `Management` [en.json `consortiumManager.sections.title`] — collapsible
  - Section `Settings` [en.json `consortiumManager.sections.section.settings`]
    - Authorization policies (Eureka; needs BE interface `policies`)
    - Authorization roles (Eureka; needs `roles`)
    - Circulation → Request cancellation reasons
    - Inventory
      - Instances: Alternative title types, Classification identifier types, Contributor types, Custom links (new, UICONSET-248), Formats, Instance note types, Instance status types, Modes of issuance, Nature of content, Resource identifier types, Resource types, Subject sources, Subject types
      - Holdings: Holdings note types, Holdings sources, Holdings types
      - Items: Item note types, Loan types, Material types
      - Instances, Holdings, Items: Statistical code types, Statistical codes, URL relationship
      - Holdings, Items: Call number types
    - Users → Departments, Patron groups (Permission sets hidden on Eureka — UICONSET-188, C566123)
  - Section `Logs & Reports` [en.json `consortiumManager.sections.section.logsAndReports`]
    - Data export (logs per member)
    - Data import (logs per member)
- Each section appears only when its BE interface is available (UICONSET-229; MODULES_REQUIRED_INTERFACES_MAP — Inventory needs any one of its interfaces, others need all).

Settings > Consortium manager (`src/settings/`) — central tenant only (C386869)
- `Network settings` [en.json `settings.heading`]
  - General → `Membership` → list `Addresses` (Name, Code, Tenant address) [en.json `settings.membership.*`]
  - `Network ordering` → `Central ordering` checkbox [en.json `settings.centralOrdering.*`]

---

## Lifecycle / Workflow

### Controlled vocabulary (Inventory / Users / Circulation)
1. Open Consortium manager (central tenant) → `Select members` → check members → Save & close.
2. Pick a setting in Management → Settings. Columns: setting fields + `Source`/`Last updated` where applicable + `Member libraries` [en.json `consortiumManager.controlledVocab.column.memberLibraries`] + Actions.
3. `+ New` (shown only when at least one member is selected AND user has `consortium-manager.edit`) → fill row → optionally check `Share` [en.json `share`] (enabled only with `consortium-manager.share`).
4. Save → modal:
   - Share checked: `Confirm share to all` / `Are you sure you want to share {term} with <b>ALL</b> members?`
   - Share unchecked: `Confirm member libraries` / `<b>{term}</b> will be saved for the member libraries:` + list of selected members
5. Confirm → PC request(s) → success toast (templated, below) or partial-failure toast per failed member.
6. Shared row: single row, Member libraries `All`, Source `consortium`, Last updated by system user. Local rows: one per selected member.
7. Share flag cannot be changed after first save (FieldSharedEntry.js disables it while editing; docs `Consortia/_index.md`). To change sharing, delete and recreate.

### Authorization roles (Eureka)
Create in the tenant chosen in the `Member` dropdown [en.json `consortiumManager.members.selection.label`] → (Central only) Actions > `Share to all` → Confirm → role type `REGULAR` → `CONSORTIUM`, Centrally managed `No` → `Yes` → edits from central propagate; member copies are read-only except Duplicate. (UICONSET-173; C523612, C523609; member-side Actions conflict — see Known Gaps)

### Publish coordinator / sharing (backend, BaseSharingService.java)
Share = for **every** consortium tenant (not only selected members): PUT if the tenant already has the record, otherwise POST; payload source set to `consortium` (settings) / `CONSORTIUM` (roles `type`, policies `source`). Delete share = DELETE in every tenant holding it; tenants where DELETE failed get PUT with source `user`/`REGULAR` (record becomes local there).

---

## Main Sections / UI Structure

### Panes
- Management [en.json `consortiumManager.sections.title`]; placeholder before a setting is chosen: "Choose settings" [C552368]
- Setting pane (title = setting label from the owning UI module, e.g. `ui-inventory.formats`)
- Authorization roles list pane: columns Name, Description, Type, Updated, Updated by (`authorization-roles/constants.js`); Member dropdown [C722382 step 1]; search
- Authorization policies list pane `Authorization policies` [en.json `authorizationPolicy.meta.title`]: columns `Name`, `Description`, `Updated`, `Updated by` [en.json `authorizationPolicy.columns.*`]; `Search` [en.json `authorizationPolicy.search`]
- Compare roles / Compare users full-screen views (two panes) (UICONSET-166/167; C552363)
- Data import / Data export log panes with `Member` dropdown (`useDataImportLogs.js`, `useDataExportLogs.js`; C397994)

### Accordions
- Role detail: Capability sets, Capabilities, Assigned users [Cypress authorizationRoles.js Accordion('Capability sets'|'Capabilities'|'Assigned users')]; General information (unverified accordion name; users are not links in CM — UICONSET-228)
- Compare: `Capabilities`, `Capability sets` / `Capabilities sets` (both keys exist in en.json: `consortiumManager.members.authorizationsRoles.roles.capabilitiesSets` vs `.capabilitiesSets`)
- Users app (cross-check): User roles [C543745], Affiliations (unverified accordion name)

### Key Navigation Paths
- Apps > Consortium manager > Select members [C386869; en.json `consortiumManager.findMember.*`]
- Consortium manager > Settings > Inventory > Instances > Subject types (any vocabulary) [CONSORTIUM_MANAGER_SECTIONS_MAP; TestRail section 51437 "View Subject types"]
- Consortium manager > Settings > Authorization roles > Actions > New / Compare roles / Compare users [en.json `…authorizationsRoles.compare`; C552363]
- Consortium manager > Logs & Reports > Data import | Data export [en.json `consortiumManager.sections.section.logsAndReports`; C552368]
- Settings > Consortium manager > Membership | Network ordering (central tenant) [en.json `settings.*`; C386869]
- User menu > Switch active affiliation (shown only when the user has >1 affiliation — `utils/checkConsortiumAffiliations.js`)
- Settings > Authorization roles (per-tenant verification of shared roles) [C523612, C722382]

---

## API Surface (mod-consortia-keycloak `consortia` 1.1 — ModuleDescriptor-template.json)

| Endpoint | permissionsRequired |
|---|---|
| GET /consortia; GET /consortia/{id}/tenants; GET /consortia-configuration | none |
| GET /consortia/{id}/_self | `*` (MODCONSKC-118: requires access token) |
| GET/POST/PUT /consortia(/{id}) | consortia.consortium.item.get/post/put |
| POST /consortia/{id}/tenants (`?adminUserId=` required) | consortia.tenants.item.post |
| PUT / DELETE / GET /consortia/{id}/tenants/{tenantId} | consortia.tenants.item.put / .delete / .get |
| POST …/tenants/{tenantId}/sync-primary-affiliations, /create-primary-affiliations | consortia.sync-primary-affiliations.item.post, consortia.create-primary-affiliations.item.post |
| POST/DELETE …/tenants/{tenantId}/identity-provider, /custom-login | consortia.identity-provider.item.post/delete, consortia.custom-login.item.post/delete |
| POST /consortia/{id}/publications; GET …/publications/{pid}; GET …/{pid}/results; DELETE …/{pid} | consortia.publications.item.post/get, consortia.publications-results.item.get, consortia.publications.item.delete |
| GET/POST/DELETE /consortia/{id}/user-tenants; GET …/user-tenants/{associationId} | consortia.user-tenants.collection.get, .item.post, .item.delete, .item.get |
| POST/DELETE /consortia-configuration | consortia.consortia-configuration.item.post/delete |
| POST/GET /consortia/{id}/sharing/instances(/{actionId}) | consortia.sharing-instances.item.post/get, .collection.get |
| POST /consortia/{id}/sharing/settings; DELETE …/settings/{settingId} | consortia.sharing-settings.item.post / .delete |
| POST /consortia/{id}/sharing/policies; DELETE …/policies/{policyId} | consortia.sharing-policies.item.post / .delete |
| POST /consortia/{id}/sharing/roles; DELETE …/roles/{rolesId} | consortia.sharing-roles.item.post / .delete |
| POST …/sharing/roles/capability-sets; DELETE …/roles/{roleId}/capability-sets | consortia.sharing-roles-capability-sets.item.post / .delete |
| POST …/sharing/roles/capabilities; DELETE …/roles/{roleId}/capabilities | consortia.sharing-roles-capabilities.item.post / .delete |

Vocabulary CRUD itself hits the owning module's endpoint in each tenant via PC (e.g. `instance-formats`, `departments`, `groups`, `cancellation-reason-storage/cancellation-reasons`, `instance-custom-links`; `src/constants.js`). Default page limit for consortia endpoints = 100 (MODCON-159).

Backend validation/errors (Karate `mod-consortia/features/tenant.feature`, `sharing-setting.feature`; swagger `schemas/tenant.yaml`):
- Tenant `code` 2–5 chars `^[a-zA-Z0-9]*$`; `name` 2–150 chars; 422 `'code' validation failed. size must be between 2 and 5`.
- Duplicate tenant → 409 `Object with code [ABC] is already presented in the system` (same pattern for id/name/isCentral; code `DUPLICATE_ERROR`).
- Unknown consortium → 404 `Object with consortiumId [<id>] was not found`.
- PUT tenant with mismatched id → 400 `Request body tenantId and path param tenantId should be identical`; central tenant cannot be deleted (`Central tenant [%s] cannot be deleted.`).
- Sharing request with mismatched ids → 400 `Mismatch id in path to settingId|roleId|policyId in request body`.

---

## Exact UI Texts

> Use verbatim. Source of truth: `translations/ui-consortia-settings/en.json` (key given). `<b>…</b>` = bold; `{x}` interpolates. Role/policy CRUD texts come from `stripes-authorization-components` (not in this repo) — cited from TestRail cases or the Cypress fragment.

### Toast Messages

**Controlled-vocabulary CRUD — one template for every vocabulary type** (`{members}` = `All` when shared, else selected/owning member names; `{count}` = 1 for shared — `SHARED_MEMBERS_COUNT`):
- Created: `<b>{term}</b> was successfully <b>created</b> for <b>{members}</b> {count, select, 1 {library} other {libraries}}.` [en.json `consortiumManager.controlledVocab.common.termCreated`]
- Updated: `<b>{term}</b> was successfully <b>updated</b> for <b>{members}</b> {count, select, 1 {library} other {libraries}}.` [en.json `…common.termUpdated`]
- Deleted: `<b>{term}</b> was successfully <b>deleted</b> for <b>{members}</b> {count, select, 1 {library} other {libraries}}.` [en.json `…common.termDeleted`]
- Partial failure create: `<b>{term}</b> was <b>not created</b> for members: <b>{members}</b>.` [en.json `consortiumManager.error.publishCoordinator.partialFailure.create`]
- Partial failure update: `<b>{term}</b> was <b>not updated</b> for members: <b>{members}</b>.` [en.json `…partialFailure.update`]
- Partial failure delete: `<b>{term}</b> could <b>not</b> be <b>deleted</b> from members: <b>{members}</b>, so the source has been changed to <b>local</b>.` [en.json `…partialFailure.delete`]
- Per-vocabulary keys also exist (`…controlledVocab.<type>.termDeleted` = `The <type> <b>{term}</b> was successfully <b>deleted</b>.`) but the generic templates above are what the controlled-vocabulary component shows unless the setting overrides `translations` — verify in env if asserting the per-type text.

| Event | Toast text | Source |
|---|---|---|
| Permission boundary (loading members without perms) | `You do not have permissions at one or more members: <b>{members}</b>` | en.json `consortiumManager.error.forbiddenMembers` |
| Role duplicated (CM) | `The role <b>{name}</b> has been successfully <b>duplicated</b>` | en.json `consortiumManager.members.authorizationsRoles.duplicate.success`; [Cypress authorizationRoles.js `The role ${roleName} has been successfully duplicated`] |
| Role duplicate error | `There was an error while duplicating the role <b>{name}</b>` | en.json `…duplicate.error` |
| Role created / updated / deleted | `Role has been created successfully` / `Role has been updated successfully` / `Role has been deleted successfully` | [Cypress authorizationRoles.js]; UICONSET-187 |
| Role create denied | `Role could not be created: Access Denied` | [Cypress authorizationRoles.js]; cases |
| Role create name collision (same tenant) | `Role could not be created: Failed to create keycloak role: name = {name}` | [Cypress authorizationRoles.js createNameErrorText]; older C523671 text had `keycloack` — see Known Gaps |
| Role shared | `Role has been shared successfully` | [Cypress authorizationRoles.js]; cases |
| Role share blocked by name collision | `Role could not be shared: Name is already in use at one or more member libraries - {member1}, {member2}.` (members bold, comma-separated) | [C692096, C1030054]; [Cypress authorizationRoles.js shareNameErrorText] |
| Role rename blocked by name collision | `Role could not be updated: Name is already in use at one or more member libraries - {member}.` | [C1263896]; [Cypress saveNameErrorText] |
| Role name with "/" | `Role name cannot contain the "/" character` | [Cypress authorizationRoles.js nameSlashErrorText] |
| Permission set saved / deleted (Okapi only) | `The permission set <strong>{permissionName}</strong> was successfully <strong>saved</strong>.` / `…<strong>deleted</strong>.` | en.json `consortiumManager.members.permissionSets.save|remove…success` |
| Member (address) deleted | `The member <b>{term}</b> was successfully <b>deleted</b>` | en.json `settings.membership.list.termDeleted` |
| Central ordering updated / failed | `Central ordering setting was successfully updated` / `Central ordering setting update failed` | en.json `settings.centralOrdering.submit.success|error.generic` |
| Authorization policy saved | `Authorization policy has been saved` | [C543762 body] — create is currently hidden in CM per code, see Known Gaps |

### Modal / Dialog Messages

| Modal | Heading / message | Source |
|---|---|---|
| Select members | `Select members`; results `Members` / `{amount, plural, one {# member} other {# members}} found`; `Total selected: {amount}`; warning `Settings for the following selected members can be modified at the same time.` | en.json `consortiumManager.findMember.*` |
| Confirm share to all (vocabularies) | `Confirm share to all` / `Are you sure you want to share {term} with <b>ALL</b> members?` | en.json `consortiumManager.modal.confirmShare.all.*` |
| Confirm share to all (roles) | `Are you sure you want to share {roleName} with ALL members?  Please note: Sharing a role with many capabilities or capability sets can take several minutes to complete, especially in systems with a large number of members. Avoid refreshing or closing this page during the process.` | [Cypress authorizationRoles.js confirmShareModalText] |
| Confirm member libraries | `Confirm member libraries` / `<b>{term}</b> will be saved for the member libraries:` | en.json `consortiumManager.modal.confirmCreate.*` |
| Delete vocabulary term | `Delete <type>` / `The <type> <b>{term}</b> will be <b>deleted</b>.` | en.json `…controlledVocab.<type>.deleteEntry|termWillBeDeleted` |
| Cannot delete (in use) | `Cannot delete the <type>` / `This <type> cannot be deleted, as it is in use by one or more records.` | en.json `…controlledVocab.<type>.cannotDeleteTerm*` |
| Duplicate role | `Duplicate role?` | [Cypress authorizationRoles.js duplicateModal]; C986307 |
| Select user (assign users) | `Select user ({tenant})` | en.json `consortiumManager.members.users.selectUser.modal.label`; UICONSET-205 |
| Switch active affiliation | user-menu `Switch active affiliation`; modal `Select affiliation`; select `Consortium members` | en.json `switchActiveAffiliation.*` |
| Central ordering activate | `Are you sure you would like to activate this functionality?`; alert `In this version of FOLIO, once activated this setting cannot be disabled.` | en.json `settings.centralOrdering.confirmModal.message`, `…alert.message` |
| Delete member (Membership) | `Delete member` / `The member <b>{term}</b> will be <b>deleted.</b>`; `Cannot delete the member` | en.json `settings.membership.list.*` |

Vocabulary `<type>` wording per key (en.json `consortiumManager.controlledVocab.<key>.deleteEntry`): alternative title type, call number type, classification identifier type, contributor type, department, format, holdings note type, holdings source, holdings type, custom link, instance note type, instance status type, item note type, loan type, material type, mode of issuance, nature of content term, patron group, resource identifier type, resource type, statistical code, statistical code type, subject source, subject type, URL relationship term.

### Button Labels (key actions)
`Select members` [en.json `consortiumManager.findMember.trigger.label`], `Confirm` [en.json `button.confirm`], `Share` (column checkbox) [en.json `share`], `New` [en.json `authorizationPolicy.new`], `Compare roles` [en.json `…authorizationsRoles.compare`], `Compare users` [en.json `consortiumManager.members.action.compareUsers`], `Share to all`, `Duplicate`, `Edit`, `Delete`, `Save & close`, `Cancel`, `Actions` (stripes / authorization components).
Tooltips: `Collapse management pane` / `Expand management pane` [en.json `collapse|expand.management.pane.button.tooltip`].

### Error / Warning Messages

| Condition | Message text | Source |
|---|---|---|
| Duplicate value in unique field (inline) | `{field} is already in use at one or more member libraries.` | en.json `validation.error.entry.duplicate` |
| Logs load failure | `Could not load jobs data.` | en.json `errors.jobs.load.common` |
| No permission (logs/policies/roles) | `User does not have required permissions.` | en.json `errors.permissionsRequired` |
| Policies load failure | `Authorization policies couldn't be loaded.` (+ `User does not have required permissions.`) | en.json `authorizationPolicy.errors.loading.data`; UICONSET-208 |
| Roles / users load failure | `Could not load authorization roles.` / `Could not load users.` | en.json `authorizationRoles.errors.loading.*`; [Cypress noAccessErrorText] |
| Compare: empty | `No capabilities found` / `No capability sets found` | en.json `…authorizationsRoles.capabilities.empty`, `…capabilitiesSets.empty` |
| Compare: placeholder | `Select authorization role`; permission-set compare `Select Member`, `Select User`, `No items to show` | en.json `…compare.placeholder`, `…permissionSets.compare.*` |
| Membership name duplicate | `Name cannot be saved. This name is already in use in the consortium` | en.json `settings.membership.error.duplicate` |
| Membership name/code length | `Tenant name must not be longer than {count} characters.` (150); `The code must not be longer than {count} characters alphanumeric.` (5); `The code must not be shorter than {count} characters alphanumeric.` (2); `The code must be unique.` | en.json `settings.membership.error.*`; `src/settings/Membership/constants.js` |

---

## Status Values

### Data import logs (CM shows only finished jobs — `useDataImportLogs.js`)
Query `statusAny = COMMITTED, ERROR, CANCELLED`; excludes `subordinationType = COMPOSITE_PARENT` and job profile `Bulk operations data import job profile-*` (UICONSET-213). Column keys: fileName, status, totalRecords, jobProfileName, startedDate, completedDate, runBy, hrId — all sortable; default sort completedDate ("Ended running" in cases) descending; page size 100 (`data-import/constants.js`). Column headers and status labels come from stripes-data-transfer-components / ui-data-import translations (not in this repo).

### Data export logs (`useDataExportLogs.js`, `data-export/constants.js`)
Query `status = COMPLETED or COMPLETED_WITH_ERRORS or FAIL`. Column keys: fileName (not sortable), status, totalRecords, exported, errors, jobProfileName, startedDate, completedDate, runBy, hrId; headers for status/total/exported/failed/startedDate use ui-data-export keys `ui-data-export.status|total|exported|failed|startedDate`; default sort completedDate descending. Failed jobs are not links (UICONSET-202); deleted job profile shown with `(deleted)` notation and file name read-only (UICONSET-242).

### Publication (PC) status — `IN_PROGRESS`, `COMPLETE`, `ERROR` (swagger `publication_schema.yaml`); per-tenant result carries `statusCode`; any 4xx/5xx tenant = failed tenant (BaseSharingService.java).

### Member tenant setupStatus — `IN_PROGRESS`, `COMPLETED`, `COMPLETED_WITH_ERRORS`, `FAILED` (swagger `schemas/tenant.yaml`); must reach COMPLETED even when the tenant has no staff users (MODCONSKC-92, MODCONSKC-135; Karate `member-tenant-setup-status.feature`).

### Record source / type of shared records (`SourceValues.java`, `src/constants.js` RECORD_SOURCE)
- Settings `source`: `consortium` (shared) → `user` on tenants where a shared delete failed; new local vocab rows default `local` (`inventory/constants.js` DEFAULT_ITEM_TEMPLATE).
- Roles `type`: `REGULAR` → `CONSORTIUM` when shared (UI `Regular` / `Consortium`; also `Default`) [Cypress constants AUTHORIZATION_ROLE_TYPES; C523612].
- Policies `source`: `CONSORTIUM` / `USER`.

---

## Capability Sets (Eureka) / Permissions

| UI permission (package.json) | Display name [en.json `permission.*`] | Eureka capability set [Cypress capability dictionaries] |
|---|---|---|
| ui-consortia-settings.consortium-manager.view | Consortium manager: Can view existing settings | Data - UI-Consortia-Settings Consortium-Manager - View |
| ui-consortia-settings.consortium-manager.edit | Consortium manager: Can create, edit and remove settings | Data - UI-Consortia-Settings Consortium-Manager - Edit |
| ui-consortia-settings.consortium-manager.share (incl. sharing-settings/roles/policies post+delete) | Consortium manager: Can share settings to all members | Procedural - UI-Consortia-Settings Consortium-Manager Share - Execute |
| ui-consortia-settings.consortia.affiliations.view / .edit | Consortia: View affiliations / Assign and unassign affiliations | Data - UI-Consortia-Settings Consortia Affiliations - View / Edit |
| ui-consortia-settings.settings.membership.view / .edit | Settings (Consortia): Can view consortia membership / Can view and edit consortia membership | Settings - UI-Consortia-Settings Settings Membership - View / Edit |
| ui-consortia-settings.settings.networkOrdering.view / .edit | Settings (Consortia): Can view network ordering / Can view and edit network ordering | (capability set name not confirmed — verify; only the permission `ui-consortia-settings.settings.networkOrdering.view` is in Cypress permissions.js) |
| Role sharing (backend) | — | Data - Consortia Sharing-Roles-All Item - Create / Delete [Cypress capabilitySets.js; Create in preconditions of C722382, C523612] |
| Custom links (Inventory settings) | — | Settings - UI-Inventory Settings Instance-Custom-Links - Manage / View [C1528153, C1528154, C1528158]; C1528159, C1554377-80 and C1528156/57 spell it `Custom-links` — spelling conflict, see Known Gaps; not in Cypress dictionary (capability set name not confirmed — verify) |

Per-domain capability sets still required in each tenant (two-layer model, UICONSET-226), confirmed names: Settings - UI-Inventory Settings Instance-Formats - View [Cypress marigoldCapabilities.js]; Settings - UI-Users Settings Departments - Manage [Cypress capabilitySets.js uiUsersSettingsDepartmentsManage]; Settings - UI-Authorization-Roles Settings - View/Create/Edit/Delete [Cypress capabilitySets.js]; Settings - UI-Authorization-Roles Users Settings - Manage [Cypress; C722382 precondition]; Settings - UI-Authorization-Policies Settings Admin - View [Cypress; C543762]; Data - UI-Data-Export - View [Cypress capabilitySets.js uiDataExportView]. Data import: the dictionary has only `Data - UI-Data-Import - Manage` (uiDataImport); a View-level set for viewing import logs is (capability set name not confirmed — verify).

---

## Settings (Settings > Consortium manager)

- Membership: list `Addresses` with `Name`, `Code`, `Tenant address`; edit via pencil (needs `consortia.tenants.item.put`); members cannot be added/removed in the UI (docs `settings_consortium-manager.md`). Name ≤150, code 2–5 alphanumeric, both unique (en.json `settings.membership.error.*`; `schemas/tenant.yaml`).
- Network ordering → Central ordering: checkbox `Allow user to select locations from other affiliations for central orders` [en.json `settings.centralOrdering.checkbox.label`]; confirmation modal on activation; once activated the checkbox is disabled (`CentralOrderingForm.js` — disabled when already true or no networkOrdering.edit). Section hidden when `orders-storage.settings` interface is absent (UICONSET-229).

---

## Common Verification Patterns

### Shared vocabulary propagation
```
Action:   Create <term> with Share checked → Confirm share to all → Confirm.
Expected: Toast "<term> was successfully created for All library." (template termCreated, count=1); single row, Member libraries "All", Source "consortium", Last updated by "System, System user - mod-consortia-keycloak"; switch active affiliation to each member → Settings > <app> > <setting> shows the term as read-only (no edit/delete) [C400671, C401723, C401725, C400666, C400669].
```

### Local vocabulary for selected members
```
Action:   Select members A,B → New → Share unchecked → Save → Confirm member libraries.
Expected: Modal lists A and B; toast "... created for A, B libraries."; one row per member; member not selected gets no row.
```

### Data import / export logs by selected member
```
Action:   Logs & Reports > Data import → keep or change Member dropdown.
Expected: Only finished jobs of that tenant (import: COMMITTED/ERROR/CANCELLED; export: COMPLETED/COMPLETED_WITH_ERRORS/FAIL); count matches the Data import/Data export app of that tenant; default sort Ended running desc; sort indicator persists when switching member.
```

### Shared role per-tenant check
```
Action:   Share role from Central → switch Member dropdown (and Settings > Authorization roles per tenant).
Expected: Role present in every tenant, Type "Consortium", Centrally managed "Yes", same capability sets; Assigned users per tenant independent (empty after share).
```

---

## Authorization Roles / Policies — Eureka Deep-Dive (Consortium Manager)

> Cypress coverage: `cypress/e2e/eureka/consortia/consortium-manager/*.cy.js` (45 specs). Cases: C523593, C523671, C523596, C552361, C552362, C552454, C552447, C986307, C692096, C1030054, C1263896, C543745, C543755, C663331, C552363, C552443, C552448, C1009047, C566123, C552368, C1045994, C543762, C523603, C523604, C523606, C523609, C523612, C722382, C850011, C877077.

### Create Role
- Roles are created in the tenant selected in the `Member` dropdown; a role created under one tenant is invisible under the others (CM list and Settings > Authorization roles) [C523671, C523593, C523596].
- Same role name may exist in different member tenants as independent local roles; editing one does not change the other [C523596].
- `New` (Actions menu of the roles list) is shown only with `consortium-manager.edit`; the create/edit routes are blocked without it or without the role/capability BE interfaces (`AuthorizationRolesViewPage.js`, `AuthorizationRolesSettings.js`); creating also requires role capabilities in the target tenant [C523665, C523582].
- `Select members` is disabled while the role (or policy) create/edit form is open (`AffiliationLookupSuppressor.js`; UICONSET-160; C523596).
- Same-name role in the same tenant fails, form stays open (toast — see Exact UI Texts) [C523671].
- Assign/Unassign opens `Select user ({tenant})`; only users of the selected tenant are listed [UICONSET-205, C523593].

### Duplicate Role
- Actions > Duplicate → `Duplicate role?` → new role name `<name> (duplicate) - <date time>`, identical capabilities/capability sets, NO assigned users, always local (not shared), in the tenant it was duplicated in — also when the source is shared (UICONSET-169; C552361, C552362, C552454, C986307).
- A duplicate of a shared role can itself be shared and deleted [C850011, C663331].

### Share to all
- `Share to all` appears only for a role selected under the Central tenant and only for users with the share capability set + Consortia Sharing-Roles-All Item (UICONSET-173, UICONSET-203; C523603). Once shared, `Share to all` is no longer offered.
- Roles created in a member tenant cannot be shared (no Share to all) (MODCONSKC-19; C523604).
- Share goes to ALL consortium tenants regardless of the `Select members` selection — even with only Central selected [C523612; BaseSharingService.java].
- Blocked case-insensitively when any member tenant already has a role with the same name; error names every conflicting member; same check on renaming a shared role (UISAUTHCOM-86; C692096, C1030054, C1263896). Fix: rename to a unique name, then share.
- Names with `&` or `#` cannot be shared after UISAUTHCOM-86 validation (UISAUTHCOM-86 / MODCONSKC-97 comments); names with `/` rejected at save [Cypress nameSlashErrorText]; CQL-like names (`and Test`, `Name==test`) share fine [C1045994].
- Capability sets missing in a member tenant: share still completes and the member role gets only the sets that exist there — C722382 example: role with `Data - UI-Consortia-Settings Consortium-Manager - View`, `Data - UI-Users - Edit`, `Settings - UI-Users Settings Addresstypes - Manage` shared to members lacking Consortium-Manager - View: members show only the Users sets and the Consortium-Manager set is NOT listed [C722382; MODROLESKC-297]. Role must be created in the Settings app, not CM (C722382 precondition).
- Sharing a large role can take minutes (modal warning); share-then-edit immediately must succeed (MODCONSKC-89 fixed "Object with publicationId [...] was not found"; C877077); CM in another tab must still load during the share (MODCONSKC-111).
- Edit of a shared role from Central propagates name/capabilities to all tenants; user assignment stays per tenant (UICONSET-173; C523609). Users can be assigned to the shared role from a member tenant [C523606].
- Shared role in a member tenant: CM Actions offer only Duplicate [C552447]; Settings > Authorization roles in a member tenant shows no Actions button [C523612] — see Known Gaps.

### Delete Role
- Actions > Delete → confirm → toast `Role has been deleted successfully`; role disappears from CM and Settings in that tenant; assigned users lose it in Users > User roles [C543745].
- Deleting a shared role from Central removes it from every tenant even with users assigned [C543755].

### Compare roles / Compare users
- Actions > `Compare roles` / `Compare users` → two panes, each with Member + Role (or User) selectors; Role selector disabled until Member chosen; changing Member clears the pane; shared capabilities/sets highlighted (UICONSET-166/167; C552363, C552448, C552443).
- Compare users lists users of the pane's tenant (request goes to that tenant — UICONSET-206), excludes patron users, loads up to 4000 users (UICONSET-243; C1009047; `USERS_LIMIT = 4000`).

### Authorization policies
- List per Member tenant with Name/Description/Updated/Updated by; filter by search (C514962, C514964, C514966).
- Create/edit/share of policies is hidden in CM: `IS_POLICIES_FEATURE_ENABLED = false` hides `New` and the create/edit routes (`AuthorizationPoliciesSettings.js`, `AuthorizationPoliciesView.js`; UICONSET-201; UICONSET-177 closed as "verify when unhidden"). Detail view still gets `displayShareAction` — see Known Gaps.
- Without policy capabilities in a tenant the list does not load and an error toast appears (UICONSET-208; C552519, C552480, C514961).

### Management Pane
- Header `Management`; collapse `<` → icon rail with `>`; focus stays on the toggle after keyboard/mouse toggle, also after duplicating a role (UICONSET-168, UICONSET-246; C1574445, C1574446, C1464337). Collapsing does not reset Select members [C552368].

---

## ECS / Multi-Tenant Notes

- Consortium manager app and Settings > Consortium manager are available in the central tenant (C386869, C397333); docs: user must be affiliated with the primary tenant to view membership.
- Two permission layers: CM-level (view/edit/share in central) + per-setting permissions in each tenant; the member list loads only data of tenants where the user has the setting permission, others produce the `forbiddenMembers` toast (UICONSET-226, UICONSET-49).
- Shared writes run as the `mod-consortia-keycloak` system user, so sharing needs only central-tenant permissions and reaches tenants the user is not affiliated with (MODCONSKC-19; C407740).
- Shared records display central-tenant metadata (Last updated = system user) because system users have different UUIDs per tenant (UICONSET-231).
- Member tenant settings: a shared value is read-only in the member's own Settings; local values created from CM are editable there (UICONSET-177 AC note; C400671 family).
- Data export logs visibility depends only on Data export permissions in the viewed tenant, not on central permissions (UICONSET-216, UICONSET-183): visible with "Data export: Can view only"/"…view logs"; hidden with only Settings (Data export) perms.
- Custom links: max 10 per tenant; `New` is disabled (C1528154/C1528157 say "disabled"; earlier note said hidden) when the limit is reached for the selection; a shared link is created only in tenants under the limit, others report failure (UICONSET-248 comments; `InstanceCustomLinks.js` LINK_LIMIT; C1528154).
- Custom links table columns in order: Name, Link text, Link, Show, Source, Last updated, Member libraries, Actions; Share checkbox sits in the Member libraries column, unchecked by default, and is disabled on inline edit of a shared link [C1528153, C1528156].
- Custom link validation (inline, New and Edit): Name — `Please fill this in to continue.`, `Name cannot be more than 150 characters.`; Link text — `Link text is required.`, `Link text cannot be more than 40 characters.`; Link — `Link is required.`, `Link must contain a valid protocol.` (no protocol and `ftp://` both rejected), `Link cannot be more than 1000 characters.`, `Link must contain a valid parameter when provided.` (static `{{ID}}`/`{{Title}}`); uniqueness of Name, Link text and Link: `<Field> is already in use at one or more member libraries.`; Cancel restores the original row value [C1554377, C1554378, C1554379, C1554380; refs UIIN-3652, UICONSET-248].
- Custom link display: Show = checked -> `View in <Link text>` under External links; a link shared via CM shows on every instance in central and member tenants; a central-Settings link only on central-context instances; a member-Settings link on that member's LOCAL instances and not on SHARED ones [C1528158, C1528155 title].

---

## Key Business Rules for Test Cases

1. Consortium manager requires ECS and is used from the central tenant; Settings > Consortium manager option is shown only in the central tenant. (C386869; docs `Consortia/_index.md`)
2. Shared vs local is chosen at save time via the `Share` checkbox and confirmed by different modals (`Confirm share to all` vs `Confirm member libraries`). (en.json `consortiumManager.modal.*`; UICONSET-215)
3. The `Share` checkbox is enabled only with `consortium-manager.share` and cannot be changed after the first save. (FieldSharedEntry.js; docs `Consortia/_index.md`; UICONSET-248)
4. `New` in vocabularies appears only when at least one member is selected AND the user has `consortium-manager.edit`. (ConsortiaControlledVocabulary.js canCreate; UICONSET-226)
5. Edit/Delete icons require `consortium-manager.edit`; for shared rows additionally `consortium-manager.share`, for local rows the setting's permission in that row's tenant. (ConsortiaControlledVocabulary.js actionSuppression/hasRequiredPerms; UICONSET-214)
6. Rows with a system source are never editable/deletable: FOLIO (classification identifier types, nature of content, resource identifier types, subject sources, subject types), rdacarrier (formats), rdacontent (resource types), rdamodeissue (modes of issuance), marcrelator (contributor types, instance status types). (getSourceSuppressor usages in `src/routes/ConsortiumManager/settings/inventory/instances/*`)
7. Departments are delete-blocked when in use (usageNumber); request cancellation reasons that require additional information cannot be deleted. (Departments.js, RequestCancellationReasons.js)
8. A shared row is a single row with Member libraries `All`; a local value appears once per selected member and not for unselected members. (UICONSET-215; en.json `consortiumManager.all`)
9. Shared value: Source `consortium`, Last updated `System, System user - mod-consortia-keycloak`. (UICONSET-231; docs `Consortia/_index.md`; Cypress CONSORTIA_SYSTEM_USER)
10. Unique fields are validated case-insensitively against the listed rows: a new or shared value must be unique across all listed members, an existing local value only within its tenant; message `{field} is already in use at one or more member libraries.` (validateUniqueness.js; en.json `validation.error.entry.duplicate`)
11. Unique fields per setting: name for most; name+code for formats, contributor types, instance status types, resource types, statistical codes, departments; group for patron groups; name+link text+link for custom links. (UNIQUE_FIELDS in each setting file)
12. Success toasts use one template per action with `{members}` = `All` for shared (count 1) or the member names; update/delete of a local row names only its tenant. (ConsortiaControlledVocabulary.js showSuccessCallout; en.json `…common.term*`)
13. When the PC fails for some tenants, a partial-failure toast names those members; a failed shared delete turns the record local (source `user`) in those tenants. (en.json `…partialFailure.*`; BaseSharingService.java updateFailedConfigsToLocalSource; Karate `sharing-patron-groups-setting.feature` "Verify source changes to user")
14. Sharing (settings, roles, policies) targets every consortium tenant regardless of the Select members selection and of the user's affiliations; existing records are PUT, missing ones POSTed. (BaseSharingService.java; C407740; C523612)
15. Loading a setting for selected members where the user lacks permission shows `You do not have permissions at one or more members: <b>{members}</b>`; other members still load. (en.json `consortiumManager.error.forbiddenMembers`; UICONSET-49; UICONSET-225/EUREKA-536)
16. Select members is disabled while a vocabulary row is being edited and while a role/policy form is open. (ConsortiaControlledVocabulary.js, AffiliationLookupSuppressor.js; C523596)
17. Each Management section appears only if its backend interface is present (Inventory: any one of its interfaces). (UICONSET-229; MODULES_REQUIRED_INTERFACES_MAP)
18. Users settings in CM show only Patron groups and Departments on Eureka; Permission sets heading is hidden. (UICONSET-188; C566123)
19. Data import logs show only COMMITTED/ERROR/CANCELLED jobs, exclude composite-parent split jobs and Bulk-edit import jobs, so counts equal the Data import app of that tenant. Data export logs show only COMPLETED/COMPLETED_WITH_ERRORS/FAIL jobs; failed jobs are not links; deleted job profiles show `(deleted)`. (useDataImportLogs.js; UICONSET-213; useDataExportLogs.js; UICONSET-202; UICONSET-242)
20. Log visibility per member depends on Data import/Data export view capability in that member only; central-tenant permissions do not grant or block it. Default sort of both log tables is Ended running descending, 100 rows per page; sort persists when the Member is changed. (UICONSET-216, UICONSET-183; C397994, C397999, C397988, C397382; data-import/constants.js, data-export/constants.js; cases)
21. Roles and policies are listed and created per tenant chosen in the Member dropdown (a role is not visible under other tenants); role `New` requires `consortium-manager.edit` plus role capabilities in the target tenant. (C523671, C523593, C523596; AuthorizationRolesViewPage.js; C523665, C523582)
22. Duplicate role: see Duplicate Role section — copies capabilities/sets, never users, always local in the current tenant. (UICONSET-169; C552361, C552362, C552454, C986307)
23. Only roles created in the Central tenant can be shared; Share to all needs the share capability set and Consortia Sharing-Roles-All Item capabilities; after sharing it is no longer offered. (MODCONSKC-19; UICONSET-173, UICONSET-203; C523603, C523604)
24. A shared role has Type `Consortium` and Centrally managed `Yes` in every tenant (member copies keep their own user assignments), is edited/deleted only from Central, edits propagate, and delete removes it everywhere even with users assigned. (C523612, C523609, C523606, C543755; UICONSET-173)
25. Share and rename of a shared role are blocked case-insensitively by a same-name role in any member tenant, naming every conflicting member. (UISAUTHCOM-86; C692096, C1030054, C1263896)
26. Role names with `/` are rejected; names with `&` or `#` cannot be shared; CQL-like names share fine. (Cypress authorizationRoles.js nameSlashErrorText; UISAUTHCOM-86 comment; C1045994)
27. Share-then-edit immediately after creation must succeed, and CM must stay usable in another tab while a large role is shared. (MODCONSKC-89/96, MODCONSKC-111; C877077)
28. Compare roles/users: two independent panes, Role/User selector disabled until Member chosen, common capabilities highlighted; Compare users excludes patrons, loads up to 4000 users, queries the selected tenant. (UICONSET-166/167/206/243; C552363, C552448, C1009047)
29. Assigned users on a role in CM are not clickable links. (UICONSET-228)
30. Authorization policies in CM are view-only in the current code: no `New`, no create/edit route. (AuthorizationPoliciesSettings.js IS_POLICIES_FEATURE_ENABLED=false; UICONSET-201)
31. Missing policy/role capabilities in a tenant produce an error toast instead of an empty list. (UICONSET-185, UICONSET-208; C552519, C552480, C503093, C503095)
32. Custom links (Inventory > Instances): max 10 per tenant; `New` disabled at the limit (C1528154, C1528157; older notes said hidden); fields Name, Link text, Link, Show, Source; shared link is created only in tenants below the limit, others get the partial-failure toast. Custom link limit: with 10 links in the selected tenant `New` is disabled; a non-shared save for that member fails with `{Name} was not created for members: {Member}`; a shared save across tenants where one is at the limit shows both the failure line for that member and the success line `{Name} was successfully created for {Member 2}, libraries` for the others (shared link is then not in the full tenant). Custom link validation messages and column order are fixed (see ECS notes) and uniqueness is checked across Name, Link text and Link independently. (UICONSET-248; InstanceCustomLinks.js; C1528154; UICONSET-248, UICONSET-249; C1554377-C1554380, C1528153)
33. Membership: name ≤150 chars and unique in the consortium; code 2–5 alphanumeric and unique; members cannot be added/removed in the UI. (en.json `settings.membership.error.*`; schemas/tenant.yaml; docs)
34. Central ordering, once activated after the confirmation modal, cannot be disabled. (en.json `settings.centralOrdering.alert.message`; CentralOrderingForm.js)
35. Switch active affiliation appears only for users with more than one affiliation; primary tenant shown with `(Primary)`. (checkConsortiumAffiliations.js; en.json `affiliation.primary.suffix`)
36. Management pane collapse/expand keeps keyboard focus on the toggle and does not reset member selection. (UICONSET-246; C1574445, C1574446, C552368)
37. Adding a member tenant must finish with setupStatus COMPLETED even with no staff users. (MODCONSKC-92, MODCONSKC-135; Karate `member-tenant-setup-status.feature`)
38. Insufficient-permission UI states (toasts, hidden actions) are expected results, not setup failures. (UICONSET-185, UICONSET-208, UICONSET-226)
39. Sharing a role to member tenants that lack some of its capability sets succeeds; the member copy contains only the sets that exist in that tenant (verified in CM per member and in Settings > Authorization roles after switching affiliation). (C722382; MODROLESKC-297, UICONSET-226)
40. After Share to all the role shows Type `Consortium`/Centrally managed `Yes` (GET returns `type: CONSORTIUM`), success toast `Role has been shared successfully`; before sharing Type `Regular`/`REGULAR`. (C523612)

---

## Existing Automated Coverage

**Karate** — `folio-integration-tests/mod-consortia/src/main/resources/thunderjet/mod-consortia/` (runner `consortia-junit.feature`, `src/test/java/org/folio/ConsortiaApiTest.java`; second runner `consortia-setup-status-junit.feature`). Features: `consortium.feature`, `tenant.feature` (CRUD, validation, soft delete/re-add), `user-tenant-associations.feature`, `consortia-self.feature`, `consortia-user-update.feature`, `consortia-user-type-update.feature`, `consortia-skip-not-required-user-types.feature`, `consortia-admin-verification-and-setup.feature`, `publish-coordinator.feature`, `sharing-instance.feature`, `sharing-setting.feature`, `sharing-patron-groups-setting.feature`, `keycloak-login-setup.feature`, `member-tenant-setup-status.feature`. No Karate coverage for sharing roles/policies or role capabilities. Shared reusables: `common-consortia/`.

**Cypress** — `stripes-testing/cypress/e2e/consortia/consortium-manager/`: access/permission specs (C397333, C397335, C397336, C386869), logs (C397382, C397988, C397994, C397999, C552358), member-side read-only checks (C400666, C400669, C400671, C401723, C401725), pane toggle (C1464337); subfolders `manage-shared-settings/` (25 settings), `manage-local-settings/` (24), `view-settings/` (27) — 192 distinct case IDs. Eureka roles/policies: `cypress/e2e/eureka/consortia/consortium-manager/` (45 specs). Fragments: `cypress/support/fragments/consortium-manager/` (consortiumManagerApp.js, per-setting fragments, data-import/data-export logs). No Cypress coverage found for Custom links in CM, Central ordering, or Membership edit.

---

## Edge Cases from Bugs

- Members list/settings sometimes did not load until reload (UICONSET-238, UICONSET-236 — Cannot Reproduce); keep a reload step out of expected results.
- Duplicating a shared role created a corrupted role without capabilities in Ramsons (UICONSET-221, known issue Ramsons; fixed via UISAUTHCOM-49) — regression candidate for duplicate-of-shared flows.
- Shared role could not be deleted (500 / `Role could not be deleted: Object with roleId [...] was not found`) due to corrupted sharing_role data after upgrade (MODCONSKC-99, not reproducible).
- Statistical code type dropdown empty when editing a local statistical code (UICONSET-200, fixed) — check edit of local codes whose type is shared.
- Shared-item result toast for members not in the current selection showed blanks/tenant IDs (UICONSET-249, open/blocked) — partial-failure toasts may show tenant IDs instead of names for unaffiliated members.
- Compare highlighting wrong if right pane is filled first (UICONSET-247, open P4).
- Session timeout while editing a role resets the Member dropdown (UICONSET-252, in refinement).
- Loan types have no Source column yet (UICONSET-240, open).
- Custom link: `New` disappears when 10 local links exist at central and a member is selected (UICONSET-248 comment, unresolved).

---

## Authoring style (measured 2026-07-23)

Consortium Manager: **`Func` 100%**, median **~18 steps** (the largest cases in the project — ~two-thirds are ≥15 steps), `User Journey` ~0%. These are big end-to-end ECS setup/administration journeys — share a setting to all members, confirm member-library selection, manage affiliations, authorization roles/policies, and verify propagation across Central + multiple member tenants. Use `Functional`. Preconditions establish the consortium (Central + ≥2 members, user affiliations, the setting/role under test); steps walk the full share/confirm/verify-per-tenant flow with a tenant-switch (its own step) before each cross-tenant assertion. Don't try to atomize these — the length is inherent to verifying propagation across tenants.

---

## Known Gaps / Items to Verify

- [ ] TestRail partially read (2026-11-25): bodies of 14 cases listed in Sources; NOT read: bodies of the other ~54 cases in sections 40125 and 137898 (e.g. C1528155, C502981, C514966, C503218 draft, C543765/C543767, C965846/965847), sections 31562, 34411, 36322 (C464308, C514991, C514999), 27637 "Data Export: Consortia" and the 76 "View/Manage shared/local" vocabulary sections under 28434. Preconditions of older cases still say `Consortium-Manager - View`/`(Since Trillium ... Edit)`.
- [ ] Authorization policies: C543762 (body re-read 2026-11-25, last updated 2026-03: New shown for Central tenant only, hidden for Member; toast `Authorization policy has been saved`; related titles C543765 edit / C543767 create local policy) conflicts with current code where create/edit are hidden for all tenants (IS_POLICIES_FEATURE_ENABLED=false, UICONSET-201). UICONSET-201 said re-enable in Sunflower; still disabled on master. Whether the policy detail view shows Edit/Share to all (`displayShareAction` passed to `PolicyDetails`) is unverified.
- [ ] Role create name collision toast: C523671 text `Failed to create keycloack role` vs Cypress fragment `Failed to create keycloak role: name = {name}`; the backend message comes from mod-roles-keycloak — confirm current text in env.
- [ ] Shared role viewed from a member tenant, both in Settings > Authorization roles: C552447 expects Duplicate present (Edit/Delete absent, pending UISAUTHCOM-20) while C523612 expects no Actions button at all. Confirm which holds (and what CM shows for a member's shared role).
- [ ] Per-vocabulary `termDeleted` keys (`The <type> <b>{term}</b> was successfully <b>deleted</b>.`) vs generic `…common.termDeleted`: which one the CM shows per setting is not confirmed (generic is used unless the setting passes its own translation).
- [ ] Data import/export status display labels come from ui-data-import / ui-data-export translations (not in this repo) — take them from those modules' en.json.
- [ ] Network ordering capability-set name: (capability set name not confirmed — verify); only permission `ui-consortia-settings.settings.networkOrdering.view` exists in Cypress permissions.js; not in capabilitySets.js.
- [ ] Custom links (UICONSET-248, Umbrellaleaf): over-limit failure toast in C1528154 is the generic partial-failure text (`{Name} was not created for members: {Member}`) while UICONSET-249 (open) says member names/IDs can be blank — confirm in env. Capability-set spelling differs across cases: `Instance-Custom-Links` (C1528153/54/58) vs `Custom-links` (C1528156/57/59, C1554377-80); neither is in the Cypress dictionary.
- [ ] Duplicate modal title: Cypress fragment `Duplicate role?` vs C552447 `Duplicate authorization role?` — confirm current text in env.
- [ ] Custom link `New` at the limit: C1528154/C1528157 say disabled, earlier UICONSET-248 comment/notes said hidden — confirm in env.
- [x] C722382 (share when capability sets missing in members): resolved 2026-11-25 from the case body — members receive only the sets that exist there; no story states it (case refs MODROLESKC-297, UICONSET-226).
- [ ] Some older TestRail preconditions still mix legacy permission names and Eureka capability sets; use the capability table above.
- [ ] Removed from this file on 2026-11-24 (out of area): Jira items UIQM/MODQM/MODELINKS/UIMARCAUTH and TestRail cases C1574452, C1563887, C1528160, C1525840, C1504468, C1504464 listed in the 2026-11-07 delta belong to marc-bib/marc-authority/requests/users areas (areas.json maps consortium-manager to UICONSET, MODCON only).
