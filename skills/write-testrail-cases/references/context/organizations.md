# FOLIO Organizations — Business Logic Context

> Built: 2026-07-21 · Refreshed: 2026-11-25 · Release: Umbrellaleaf R2 2026 (ui-organizations master 7.1.0 in progress; module versions: Trillium R1 2026 = ui-organizations 7.0.0) · Builder: build-app-context (refresh)

> Sources (refresh 2026-11-24):
> - GitHub: `folio-org/ui-organizations` master 7.1.0 (`translations/ui-organizations/en.json`, `package.json`, `src/**`, `CHANGELOG.md`); `mod-organizations` (`descriptors/ModuleDescriptor-template.json`, `ErrorCodes.java`, `ProtectionServiceImpl.java`, `NEWS.md`); `mod-organizations-storage` (ModuleDescriptor, `db_scripts/schema.json`, `NEWS.md`); `acq-models` `mod-orgs/schemas/organization.json`.
> - Jira (folio-org, anonymous REST): UIORGS, MODORG, MODORGSTOR Done/Closed stories+bugs resolved since 2024-06 (44 issues) + open UIORGS-485/487, MODORG-78..81.
> - TestRail project 14, suite 21: section 107 "Organizations" (74 cases), 121 Settings (10), 201 Interface details (8), 31406 Consortium (1); C350552 from EDIFACT export section 17493. Release field: 21 = Umbrellaleaf, 20 = Trillium, 19 = Sunflower, 18 = Ramsons.
> - folio-integration-tests (`acquisitions/.../thunderjet/mod-organizations`), stripes-testing (`cypress/e2e/organizations`), docs.folio.org source (`content/en/docs/Acquisitions/Organizations/_index.md`, `Settings/Settings_organizations`), Confluence "Sunflower UAT - Organization version history".
> UI texts below are from `en.json` (key named) or a TestRail case (C-ID). `en_US.json` matches `en.json` except the missing key `integration.edi.accountNumbers.inUseWarning` (2026-11-24). `<b>` in a string renders bold.

---

## What is Organizations

The Organizations app holds records for **vendors** and other external parties (donors, access providers, licensors) that the acquisitions apps reference. An organization record carries contact info, contact people, interfaces (platform admin logins), agreements, vendor-specific settings (payment/claiming/intervals/tax), EDI/CSV **integration** configs (automated order export and claims export), accounts, and banking information. Orders, Invoices, Receiving, Agreements and eUsage link to organizations. Orders can only be opened and invoices paid when the organization is a vendor with status Active (docs.folio.org Organizations › Summary).

---

## Key Terms

| Term | Definition |
|---|---|
| Organization | The record; may be flagged `Vendor` and/or `Donor`. Required: `name`, `code`, `status` (acq-models `organization.json`). |
| Vendor | An organization the library buys from (orders/invoices reference it). Unlocks Vendor information, Vendor terms, Accounts, Banking information, Integration details. |
| Interface | A platform/admin endpoint with URI + credentials (for e-resource stats etc.); separate record in `organizations-storage/interfaces`, linked by ID. |
| Integration | Export config for the vendor, type **Ordering** (EDIFACT order export) or **Claiming** (claims export, CSV or EDI). Stored in mod-data-export-spring (`data-export-spring/configs`), not in mod-organizations (UI `useIntegrationConfigMutation.js`; MODORG-73/MODORGSTOR-171 closed Won't do). |
| Account | A vendor account number the library uses; must be unique within the org. |
| Accounting code | External accounting-system code (`erpCode`) used on invoice export. |
| Privileged contact | Donor contact stored in `organizations-storage/privileged-contacts`, separate from regular contacts. |

---

## Record structure — accordions

`Summary`, `Contact information`, `Contact people`, `Privileged donor information` (only when `Donor` is checked and user can view it), `Interface`, then vendor-only: `Vendor information`, `Vendor terms`, `Integration details` (view only), `Accounts`, `Banking information` (tenant setting + permission), and `Agreements` (linked ERM agreements, view). Labels: en.json keys `summary`, `contactInformation`, `contactPeople`, `privilegedDonorInformation`, `interface`, `vendorInformation`, `vendorTerms`, `integrationDetails`, `accounts`, `bankingInformation`, `agreements`. Source: `OrganizationDetails.js`, `OrganizationForm.js`.

**Summary fields:** `Name`, `Code`, `Description`, `Accounting code`, `Default language`, `Alternative names` (`Alias`, required once a row is added), `Organization status` (`Active` / `Inactive` / `Pending`), `Vendor`, `Donor`, `Type`.
**Vendor information:** `Payment method`, `Vendor currencies`, `Claiming interval`, `Discount percent`, `Expected activation interval`, `Expected invoice interval`, `Expected receipt interval`, `Renewal activation interval`, `Subscription interval`, `Tax`, `Tax ID`, `Tax percentage`, `Liable for VAT`, `Export to accounting`.
**Accounts:** `Name`, `Account number` (required), `Description`, `Accounting code`, `Payment method`, `Account status`, `Contact info`, `Library code`, `Library EDI code`, `Notes`, acquisition units.
**Banking information:** `Bank name`, `Bank account number`, `Transit number`, `Address category`, `Account type`, `Notes`, `Use as primary banking information` (en.json `data.bankingInformation.*`); button `Add banking information`.
**Results list columns:** Name, Code, Description, Status, Vendor (sortable) — `OrganizationsList.js`.

### Search & filter
- Search options (en.json `search.*`): `All`, `Name`, `Code`, `Language`, `Alias`, `Accounting code`, `Tax ID`. Placeholder for Language: `Search with language code`.
- `All` = OR of name/code/language/aliases/erpCode/taxId with prefix match, **plus** `bankingInformation.bankAccountNumber` only if the user has banking view permission (`useBuildQuery.test.js`; C423426, C423427, C423432).
- Filters (en.json `filterConfig.*`): `Organizations status`, `Types`, `Tags`, `Is donor`, `Is vendor`, `Country`, `Languages`, `Payment method`, `Acquisition unit`, `Address category`, `Contact people category`, `Stats available`, `Created by`, `Date created`, `Updated by`, `Date updated`.

---

## Status Values

- **Organization status:** `Active`, `Inactive`, `Pending` (acq-models enum; en.json `organizationStatus.*`). Required.
- **Contact people status:** `Active`, `Inactive` (en.json `contactPeople.status.*`).
- **Organization type status (Settings > Types):** `Active`, `Inactive` (en.json `settings.typeStatus.*`).
- **Integration type:** `Ordering` (export type `EDIFACT_ORDERS_EXPORT`), `Claiming` (export type `CLAIMS`) — `OrganizationIntegration/constants.js`.
- **Transmission method:** `File download`, `FTP`. **File format:** `CSV`, `EDI`. **EDI FTP:** `SFTP`, `FTP`. **FTP mode:** `ASCII`, `Binary`. **FTP connection mode:** `Active`, `Passive`.
- **Vendor/Library EDI type:** `31B (US-SAN)`, `014 (EAN)`, `091 (Supplier-assigned ID)`, `092 (Library-assigned ID)` (`constants.js`).
- **Schedule period:** `Hourly`, `Daily`, `Weekly`, `Monthly`; placeholder `Please select schedule period` (en.json `integration.scheduling.schedulePeriod.*`).
- **Linked agreement status (Agreements accordion):** `Active`, `Closed`, `Draft`, `In negotiation`, `Requested`.

---

## Exact UI Texts

### Toasts
| Event | Text | Source |
|---|---|---|
| Organization saved | `The <b>Organization - "{organizationName}"</b> has been successfully <b>saved</b>` | en.json `save.success` |
| Organization deleted | `The organization <b>{organizationName}</b> was successfully <b>deleted</b>` | en.json `organization.delete.success` |
| Contact saved / unassigned / deleted | `The contact was saved` / `The contact was unassigned` / `The contact was deleted` (fail: `The contact was not saved` / `… not unassigned` / `… not deleted`) | en.json `contacts.message.*` |
| Interface saved | `The interface was saved` (fail: `The interface was not saved`, `The interface was not unassigned`, `The interface was not deleted`) | en.json `interface.message.*` |
| Integration saved / save failed | `Integration was saved` / `Integration could not be saved` | en.json `integration.message.save.*` |
| Integration deleted / delete failed | `The integration was successfully <b>deleted</b>` / `The integration was not <b>deleted</b>` | en.json `integration.message.delete.*` |
| Integration duplicated / failed | `Integration <b>{term}</b> was successfully duplicated` / `Integration was not duplicated` ({term} = `<original name> (<date, time>)`, C688767) | en.json `integrationDetails.duplicate.*` |
| Banking info save failed | `Failed to save banking information.` | en.json `bankingInformation.save.error` |
| Settings saved | `Setting was successfully saved` (number generator) / `Setting was successfully updated.` (account types) | en.json `settings.numberGeneratorOptions.save.success`, `settings.accountTypes.save.success.message` |
| Category / type / account type deleted | `The category <b>{term}</b> was successfully <b>deleted</b>` (same pattern for type, account type) | en.json `settings.*.termDeleted` |

### Create / delete / confirm modals
| Modal | Text | Source |
|---|---|---|
| Create / edit page title | `Create organization` / `Edit - {name}` | en.json `createOrg.title`, `editOrg.title` |
| Delete organization | heading `Delete {organizationName}?`, message `Delete organization?`, buttons `Cancel` / `Delete` | en.json `organization.delete.*`, `view.delete.message`; C613152 |
| Vendor-data deletion warning (uncheck Vendor in Edit) | `Confirm vendor data deletion` / `Warning. All vendor information, vendor terms, EDI information, and accounts will be deleted from this record.` → `Confirm` | en.json `vendor.confirmation.*`; C358965 |
| Unassign contact / interface | `Unassign from organization` / `Are you sure you want to unassign the contact from this organization?` (interface: `…unassign the interface from…`) | en.json `contacts.confirmUnassign.*`, `interface.confirmUnassign.*` |
| Delete contact / interface | `Delete contact` / `Are you sure you want to delete the contact?`; `Delete interface` / `Are you sure you want to delete the interface?` | en.json |
| Delete integration | `Confirm integration deletion` / `Are you sure you want to delete the integration?` → `Confirm` | en.json `integration.confirmation.*` |
| Duplicate integration | `Duplicate integration config` / `Are you sure you want to duplicate the <b>{term}</b> integration config?` → `Cancel` / `Confirm` | en.json; C688767 |
| Contact / interface pane titles | `Create contact`, `Edit contact - {name}`, `Contact - {name}`; `Create interface`, `Edit interface - {name}` | en.json |
| Integration page titles | `Create integration`, `Edit integration - {name}`; button `Add integration` | en.json `integration.create*` |

### Errors / validation
| Condition | Message | Source |
|---|---|---|
| Org save (generic / unknown backend code) | `Organization could not be saved` | en.json `save.error.genericError`; `handleSaveErrorResponse.js` |
| Code duplicate | `Code is already in use` | en.json `save.error.codeInUse`; `validateOrgCode.js` |
| Duplicate account numbers (form) | `An organization can not have duplicate account numbers` | en.json `save.error.accountNumberMustBeUnique` |
| Duplicate account numbers (view banner) | `An organization can not have duplicate account numbers. Before editing this organization in any way you will need to change or remove any duplicate account numbers` | en.json `view.duplicateAccounts` |
| Credentials save / interface assign | `Error while saving credentials` / `Error while assigning interface to the organization` | en.json |
| No primary record | `No primary record selected` | en.json `contactPeople.errors.noPrimaryRecord` |
| Required field | `Required!` | C688767 (stripes-acq-components text) |
| Integration account number in use (form + view banner) | `The following account numbers are also assigned to another integration of the same type: {accounts}. An account number should be assigned to only one integration.` ({accounts} = `123 (Name A, Name B); 456 (Name C)`) | en.json `integration.edi.accountNumbers.inUseWarning`; `utils.js` |
| Monthly `Day` field | `Required` / `Value must be less than or equal to 31` / `Value must be greater than or equal to 1` | C434063 |
| Account type duplicate | `Account type must be unique.` | en.json |
| Settings generic | `Something went wrong.` / `Error saving setting` | en.json |

### Integration form — key fields
`Integration information` accordion: `Integration name` (required), `Description`, `Default integration`, `Integration type`, `Transmission method`, `File format`, `Account numbers` (multi-select). `EDI configuration`: `Automate order export for acquisition methods`, `Vendor EDI code`, `Vendor EDI type`, `Library EDI code`, `Library EDI type`, `EDI naming convention` (always disabled; default `{organizationCode}-{integrationName}-{exportJobEndDate}`), `Send account number`, `What messages are expected for this vendor?` (`Orders` / `Invoices`), `Notes`. `FTP details`: `EDI FTP`, `FTP mode`, `Server address`, `FTP connection mode`, `Username`, `Password`, `FTP port`, `Order directory`, `Invoice directory`, `Notes`. `Scheduling` (Ordering only): `Schedule EDI`, `Schedule period`, `Schedule frequency`, `Date`, `Time`, weekday checkboxes, `Day`. Source: en.json `integration.*`, `EdiForm.js`, `utils.js` `getDefaultEdiNamingConvention`.

### Settings
`Categories`, `Types` (delete guard: `Cannot delete category` / `This category cannot be deleted, as it is in use by one or more records.`; `Cannot delete type` / `This type cannot be deleted, as it is in use by one or more records.`), `Banking information` (`Enable banking information`), `Account types` (`Account type must be unique.`; delete guard below), `Number generator options` — heading `Code` with a drop-down `Off` / `On, field editable` / `On, field not editable` (radio buttons replaced in 7.0.0, UIORGS-462). Settings pane title `Organizations settings`. Source: en.json `settings.*`.

---

## API Surface (Eureka/Okapi, permissionsRequired)

**mod-organizations** (business logic, acq-unit protection, account-number uniqueness):
| Endpoint | Methods | Permission |
|---|---|---|
| `/organizations/organizations` | GET, POST | `organizations.organizations.collection.get`, `…item.post` |
| `/organizations/organizations/{id}` | GET, PUT, DELETE | `organizations.organizations.item.get` / `.put` / `.delete` |
| `/organizations/banking-information[/{id}]` | GET, POST, GET, PUT, DELETE | `organizations.banking-information.collection.get`, `.item.post/get/put/delete` |

**mod-organizations-storage**: `/organizations-storage/{organizations, contacts, privileged-contacts, interfaces, organization-types, categories, banking-account-types, banking-information, settings}` with CRUD (`organizations-storage.<resource>.collection.get`, `.item.post/get/put/delete`); `/organizations-storage/interfaces/{id}/credentials` GET/POST/PUT/DELETE (`organizations-storage.interfaces.credentials.item.*`); timer `POST /organizations-storage/audit-outbox/process`. Banking-information tenant switch = settings record `key=BANKING_INFORMATION_ENABLED`.

**Integrations**: `data-export-spring/configs[/{id}]` (mod-data-export-spring). UI sends `exportTypeSpecificParameters.vendorEdiOrdersExportConfig.*` and `type` = `EDIFACT_ORDERS_EXPORT` or `CLAIMS`.

**Backend error codes** (`mod-organizations` `ErrorCodes.java`): `accountNumberMustBeUnique` (422), `organizationCodeWhitespace` (422), `organizationAcqUnitsNotFound`, `userHasNoAcqUnitsPermission` (403), `userHasNoPermission` (403), `idMismatch`, `genericError`. Storage delete of an org still referenced by banking info used to return `genericError` "…still referenced from table banking_information" (MODORGSTOR-172, fixed 4.8.1).

---

## Capability Sets (Eureka)

> Names confirmed by TestRail preconditions (Eureka block of C423547, C423630, C423502, C367989/C367990) and docs.folio.org (Organizations › Capabilities & capability sets; Settings_organizations), types/resources/actions cross-checked with stripes-testing `cypress/support/dictionary/capabilitySets.js` (`UI-Organizations` view/delete only) and ui-organizations `package.json` permission names. Cases spell them `Data - UI-Organizations - View` (Type - Resource - Action, each capitalised); legacy display names (en.json `permission.*`, `permissions.js` `gui`) shown for mapping only.

| Action | Capability set (Type - Resource - Action) | Legacy display name | Confirmed by |
|---|---|---|---|
| View | `Data - UI-Organizations - View` | `Organizations: View` | C423547; docs; Cypress `uiOrganizationsView` |
| View, edit | `Data - UI-Organizations - Edit` | `Organizations: View, edit` | C423547; C423630; docs |
| View, edit, create | `Data - UI-Organizations - Create` | `Organizations: View, edit, create` | C423547; docs |
| View, edit, delete | `Data - UI-Organizations - Delete` | `Organizations: View, edit, delete` | C423547; docs; Cypress `uiOrganizationsDelete` |
| Interface credentials view / manage | `Data - UI-Organizations Creds - View` / `Data - UI-Organizations Creds - Manage` | `Organizations: Interface usernames and passwords: view` / `…: view, edit, create, delete` | C423547; docs |
| Integration credentials view / edit | `Data - UI-Organizations Integrations Creds - View` / `Data - UI-Organizations Integrations Creds - Edit` | `Organizations: Integration usernames and passwords: view` / `…: view, edit` | C423547; docs |
| Banking info view / edit / create / delete | `Data - UI-Organizations Banking-Information - View` / `- Edit` / `- Create` / `- Delete` | `Organizations: View banking information` … `…View, edit, create and delete banking information` | C423547; docs |
| Privileged donor view / edit | `Data - UI-Organizations Privileged-Contacts - View` / `Data - UI-Organizations Privileged-Contacts - Edit` | `Organizations: can view privileged donor information` / `…can view, create, edit, delete…` | C423630 (Edit); docs (View, Edit) |
| Assign acq units on create | since Trillium `Procedural - UI-Organizations Organizations Acquisitions Units Assign - Execute` (before: `Procedural - UI-Organizations AcqUnits - Execute`) | `Organizations: Assign acquisition units to new organization` | C423547 |
| Manage acq units on edit | since Trillium `Procedural - UI-Organizations Organizations Acquisitions Units Manage - Execute` (before: `Data - UI-Organizations AcqUnits - Manage`) | `Organizations: Manage acquisition units` | C423547 |
| Settings view | `Settings - UI-Organizations Settings - View` | `Settings (Organizations): View settings` | C423547; docs Settings_organizations |
| Settings view and edit | `Data - Organizations Settings - Manage` (no `UI-` prefix) | `Settings (Organizations): Can view and edit settings` | C367989, C367990, C423547; docs |
| Number generator options | `Settings - UI-Organizations Settings NumberGenerator - Manage` | `Settings (Organizations): Manage number generator options` | docs Settings_organizations + service_interaction; package.json `ui-organizations.settings.numberGenerator.manage` (no TestRail precondition yet) |

Notes: the acq-unit renames come from UIORGS-478 (7.0.0, Trillium; C423547 preconditions); docs.folio.org Organizations still lists the pre-Trillium `UI-Organizations AcqUnits` resource (docs lag behind). `Organizations: Integration usernames and passwords: view` alone does not give access to organization records (C423502: only the Finance app shows in the header). Interface credential and privileged-donor edit sets must be combined with `UI-Organizations` create or delete (docs.folio.org).

---

## Common Verification Patterns

### Create a vendor
1. Actions → `New`; fill `Name`, `Code`, set `Organization status`, check `Vendor` → `Save & close`.
2. Expected: toast `The Organization - "{name}" has been successfully saved`; record shows in list with the code; `Code is already in use` under Code if the code collides.

### Add an interface with credentials
Interface accordion → `Add interface` (find-interface lookup) or `New` (create) → `Name`, `URI`, `Username`, `Password` → Save (`The interface was saved`). Password masked by default; `Show credentials` / `Hide credentials` (C3455). Credentials gated by `UI-Organizations Creds`.

### EDI integration for automated order export
Integration details → `Add integration` → `Integration type = Ordering`, account numbers, EDI codes, FTP details, `Schedule EDI` period → `Save & close` (`Integration was saved`).

### Verify a scheduled EDI export actually ran (C965832, C1621423)
Don't stop at saving the schedule: open an eligible Order (Actions → Open), wait for the scheduled time, then **Export Manager** → `Organizations` toggle → `Integration type` = `Orders` → job in `Successful` status for each integration. A case that only checks `Integration was saved` doesn't prove the export happens.

### Verify an API-backed rule in two places
For timezone (Rule 11) and account-number rules, check both the UI (form/view) and the stored config (`GET data-export-spring/configs/{id}` → `vendorEdiOrdersExportConfig.ediConfig.accountNoList`, `ediSchedule.scheduleParameters`).

---

## "Save & Keep Editing" Button (Create/Edit Organization) [C656329, C656335; UIORGS-449]

Three buttons: `Cancel` (always active), `Save & keep editing` (disabled until ≥1 field filled), `Save & close`. Available on Create, Edit and Clone (UIORGS-449 title; C656335 covers Edit). Save & keep editing runs the same validation without closing; on success shows the save toast and focus stays in the last-clicked field.

## Version History (Organizations) [C663330; UIORGS-355/356/359/482]

Clock icon → fourth `Version history` pane; Actions button disappears from the details pane; Version history and Tags icons become inactive; cards newest-first with `MM/DD/YYYY, HH:MM AM` link; oldest = "Original version", newest = "Current version" + "Changed" list; changed fields highlighted yellow. Organizations nuances:
- **Privileged donor information and Agreements accordions are never shown in any version card** (C663330).
- Each version shows the accordion contents at that time: deleted Contact people/Interface entries show "The list contains no items" while "Interface"/"Contact people" appear in "Changed" (C663330).
- Checking `Vendor` is tracked like any field; Vendor-only accordions appear for that version.
- Metadata fields (`Created by`, `Created date`, `Updated by`, `Updated date`) are not listed as changed fields (UIORGS-482, ui-organizations 7.1.0). A save that changes nothing but metadata appears as a card "Changed: System maintenance update" (UIORGS-482 QA comment, 2026-11; text not in ui-organizations en.json — verify).
- Edit audit events now carry the pre-edit snapshot (`originalOrganizationSnapshot` + `organizationSnapshot`; create has only `organizationSnapshot`) — MODORGSTOR-201; Karate `cross-modules/features/audit-event-organization.feature`.

## Number Generator for Organization Code [C700855; UIORGS-336/337/462]

Settings > Organizations > Number generator options, heading `Code`, drop-down:
- **Off**: no `Generate vendor code` button on Create or Edit; code entered manually.
- **On, field not editable**: `Generate vendor code` under Code opens `Vendor code generator` to pick a sequence; Code then cannot be edited manually.
- **On, field editable**: same flow, Code stays editable (prefix/suffix allowed) or can be typed manually.
- Requires a sequence under Settings > Service interaction > Number generator sequences for generator code `organizations_vendorCode` ("Organizations: Vendor code") and the NumberGenerators role (C700855). Settings page needs `UI-Organizations Settings NumberGenerator - manage` (docs).
- Re-generating on Edit overwrites the previous Code.

## Integration Create/Edit — Dynamic Field Requirements [C688767, C825311, C434063; UIORGS-442, UIORGS-468; `utils.js`, `IntegrationInfoForm.js`]

- Default on Create: `Integration type = Claiming`, `Transmission method = File download`, `File format = CSV`; Account numbers optional; no EDI/FTP field required; **Scheduling hidden**.
- `Integration type = Ordering` sets and disables `Transmission method = FTP` and `File format = EDI` (CSV and File download not available), sets export type `EDIFACT_ORDERS_EXPORT`, and shows **Scheduling** (`Schedule EDI` unchecked). Switching back to Claiming re-enables both dropdowns keeping their current values, sets export type `CLAIMS`, hides Scheduling and **clears the saved schedule** (`ediSchedule = null`).
- **File format alone drives EDI requiredness**: EDI ⇒ `Account numbers`, `Vendor EDI code`, `Library EDI code` required; CSV ⇒ not required.
- **Transmission method alone drives FTP requiredness**: FTP ⇒ `Server address` (must be a valid URL, `validateURLRequired`) and `FTP port` required; File download ⇒ not required.
- Validation re-evaluates immediately after changing type/method/format (UIORGS-468, 6.0.1); before the fix it applied to the previous option.
- Scheduling: `Schedule period` cannot stay `Please select schedule period`; `Schedule frequency` positive number; `Time` required; `Day` (Monthly) numeric 1–31 (`SchedulingForm.js`; C434063, C434064).
- `Default integration` checkbox is disabled when another integration of the org is already default; a default integration has no `Account numbers` field (`IntegrationInfoForm.js`). Docs: use it when the vendor has no account numbers.
- `EDI naming convention` is read-only (C350758).
- Token refresh while the form is open must not change the type: UIORGS-472 (fixed 7.0.0) — UI sent `Claims` instead of EDIFACT after refresh and save failed with `Integration could not be saved`; regression case C825311.
- On save the UI also stores `vendorName` = organization name in the config (UIORGS-485, In Review on 2026-11-24; already done on Duplicate in code).

## Integration Account Numbers [UIORGS-271, UIORGS-481; `utils.js` `buildAvailableAccounts`, `AccountNumbersInUseBanner.js`]

- `Account numbers` lists the org's accounts minus those used by **another integration of the same type** (Ordering vs Claiming are separate since UIORGS-481; before, the exclusion was cross-type).
- Account numbers stored on the integration being edited always stay listed and selected (and can be deselected), even if another integration also uses them or the account no longer exists on the org.
- If another same-type integration uses a selected number, a warning banner (`integration.edi.accountNumbers.inUseWarning`) shows in the edit form and in the integration view; not shown for a default integration.
- Rationale for one-account-one-integration: prevent an order from being exported twice (UIORGS-271 comment, Dennis Bridges).

## Duplicate Integration [C688767; UIORGS-441; `OrganizationIntegrationView.js`]

Integration view → Actions → `Duplicate` → modal `Duplicate integration config` → `Confirm` → toast `Integration <name> (<date, time>) was successfully duplicated`; the copy is named `Copy of <name> (<date, time>)` and returns to the org view with two integrations. The copy carries every field (FTP credentials, schedule, **and account numbers** — the whole config is cloned minus `id`). After UIORGS-481 the copy's edit form shows its stored account numbers with the in-use warning (the original also warns); before UIORGS-481 the field appeared empty in both (C688767 step 19 still expects "NOT displayed" — see Known Gaps).

## Deprecated Acquisition Methods in EDI Configuration [UIORGS-487 (In QA 2026-11-24), C1621423, ui-organizations 7.1.0 CHANGELOG]

- `Automate order export for acquisition methods` does not offer methods marked Deprecated in Settings > Orders > Acquisition methods for a new selection (create and edit).
- Methods already saved on the integration stay listed and selected with the suffix `(deprecated)` in view and edit; the integration saves unchanged and its scheduled export still runs (C1621423 steps 1–10).
- A deselected deprecated method stays in the list until save/cancel; after saving without it, it is no longer offered (C1621423 steps 11–14).
- System methods show translated names, same as in Orders (e.g. `Approval plan`, `Purchase at vendor system`).

## Privileged Donor Contacts [C423630, C423618, C423690, C423623–C423632]

- Accordion shows only when `Donor` is checked and the user has `UI-Organizations Privileged-Contacts - view` (`OrganizationDetails.js`, `OrganizationForm.js`); available on vendor and non-vendor orgs.
- Donor contacts added via `Add donor` > `New` never appear in the `Contact people` `Add contact` lookup — `0 records found` / `No results found for "{query}". Please check your spelling and filters.` even for a user with full rights (C423630).

## Banking Information [C423547, C423503–C423519, C613152, C434070; UIORGS-388/389/436, MODORGSTOR-172]

- Shown only for vendors, only when Settings > Organizations > Banking information > `Enable banking information` is checked, and only with `UI-Organizations Banking-Information - view` (edit page: `- edit`). Tenant setting off ⇒ accordion absent for everyone (C423547). Banking changes are skipped on save when the org is not a vendor or the setting is off (`useBankingInformationManager.js`).
- Settings > Banking information is hidden from users without banking view permission (UIORGS-436).
- A user without banking permissions editing an org gets no banking error (C434070).
- Deleting an organization that has banking information deletes the banking records too and succeeds (MODORGSTOR-172; C613152).

## Bank Account Type — Delete Guard [C590820; UIORGS-435, MODORGSTOR-163]

Settings > Organizations > Account types: delete opens `Delete account type` (`The account type <b>{term}</b> will be <b>deleted.</b>`, Cancel/Delete); if the type is used by any organization's banking information, a second popup `Cannot delete account type` / `Unable to delete. Account type is in use by one or more organizations.` with `Okay` appears and nothing is deleted. Account type names are unique (DB unique index on `banking_account_types.name`; `Account type must be unique.`).

## Vendor / Donor Toggle — Side Effects [C730, C358965, C421981; `OrganizationSummaryForm.js`]

- Checking `Vendor` in Edit reveals Vendor information / Vendor terms / Accounts (and Banking information, if enabled and permitted) at once; Integration details exists only on the view pane, so it appears after save.
- Unchecking `Vendor` on an **existing** org opens `Confirm vendor data deletion`; `Confirm` clears all vendor fields and hides the three accordions; cancelling restores `Vendor` checked. No dialog on Create (new record).
- `Donor` is read-only on the View pane; toggle it in Edit. Unchecking `Donor` drops the org from an active `Is donor = Yes` filter at once.

## "Add Contacts" / "Add Donors" Picker [C359169, C422163]

50 records per page; header "Select all" selects only the current page; "Total selected" persists across Next/Previous; deselecting one row unchecks "Select all"; `Save` active once ≥1 record selected.

---

## ECS / Multi-Tenant Notes

- Only one ECS case: C423433 "ECS | Searching in "Organization look-up" by "Bank account number" without banking permissions" (section 31406).
- Organizations are tenant-local. ECS reference data creates the same vendor and donor organizations in central and every member tenant (MODORGSTOR-197, 5.0.0 Trillium) — without integration details.
- Shared/consortial organizations (share endpoint, central-only update, PATCH of shadow copies, `shared` flags) are **not implemented**: MODORG-78/79/80/81 and MODORGSTOR-174 are In Refinement (2026-11-24). Do not write cases for sharing.

---

## Existing Automation Coverage

- **Karate** (folio-integration-tests, `acquisitions`): `thunderjet/mod-organizations/features/acquisitions-api-tests.feature` (acq-unit read protection: not protected / read-open / full-protected orgs before and after unit membership; POST/PUT with duplicate account numbers → 422), runner `OrganizationsApiTest.java`; `thunderjet/cross-modules/features/audit-event-organization.feature` (Create/Edit events, 50 updates); `thunderjet/mod-invoice/features/check-that-can-not-approve-invoice-if-organization-is-not-vendor.feature`; integration configs via `thunderjet/mod-data-export-spring/reusables/create-integration-details.feature` (EDIFACT and claims export features). Not covered by Karate: code whitespace (MODORG-99), banking-information API, privileged contacts, interface credentials.
- **Cypress** (stripes-testing): `cypress/e2e/organizations/*.cy.js` (65 specs incl. `integration-claiming`, `integration-day-field-validation`, `verify-two-integrations-cannot-have-same-account-number` (C350552), `version-history-view`, banking/privileged-donor permission specs, `save-and-keep-editing-*`), `cypress/e2e/organizations/interface-details/` (7), `cypress/e2e/organizations/settings/` (3), `cypress/e2e/settings/organizations/` (2); fragments `cypress/support/fragments/organizations/` (`integrations/integrationEditForm.js`, `versionHistorySection.js`, `organizationDetails.js`).

---

## Key Business Rules for Test Cases

1. An organization is a **vendor** only when `Vendor` is set; only vendors are selectable in Orders/Invoices vendor fields, and orders open / invoices pay only for an Active vendor. (github + docs.folio.org; Karate `check-that-can-not-approve-invoice-if-organization-is-not-vendor.feature`)
2. **Organization code must be unique** — async check `code=="<code>" and id<>"<id>"` shows `Code is already in use`; DB unique index backs it. (`validateOrgCode.js`; storage `make_organization_code_unique.sql`)
3. **Account numbers must be unique within an organization** — form error `An organization can not have duplicate account numbers`; API POST/PUT returns 422 `accountNumberMustBeUnique`; a record already holding duplicates shows the `view.duplicateAccounts` banner. (`FieldAccountNumber.js`; Karate `acquisitions-api-tests.feature`; C350761)
4. Unchecking `Vendor` on an existing organization in Edit triggers `Confirm vendor data deletion`; Confirm removes vendor info, vendor terms, EDI info and accounts; deleting the whole organization uses the separate `Delete {organizationName}?` modal. (`OrganizationSummaryForm.js`; C358965, C613152)
5. Interface and integration **credentials are separately permissioned**; integration-creds view alone gives no access to organization records. (`package.json` permissionSets `ui-organizations.creds.*` / `ui-organizations.integrations.creds.*`; docs.folio.org Organizations capabilities; C423502)
6. The vendor's `Claiming interval` (days) is the default claiming interval on purchase order lines for that vendor (overridable on the POL and on the Receiving title; the POL value is inherited by the title); it defines how long after a piece's expected receipt date it becomes `Late`. Other vendor intervals (expected receipt / invoice / activation) are stored on the vendor only (not verified how Orders/Invoices use them). (docs.folio.org Organizations › Vendor information, Receiving › Claiming; Karate `thunderjet/mod-orders/features/poline-claiming-interval-checks.feature` for the POL → title inheritance)
7. Integration config (type Ordering/Claiming, FTP, schedule) drives automated EDIFACT order export and claims export via Export Manager. (UIORGS-442; `OrganizationIntegration/constants.js`; Cypress `integration-claiming`)
8. Contacts and interfaces can be **unassigned** (link removed, record kept) or **deleted** (record removed). (en.json `contacts.confirmUnassign.*`, `contacts.confirmDelete.*`)
9. Acquisition units gate org access like other acq records; assign-on-create and manage-on-edit are distinct capabilities; changing `acqUnitIds` on update without manage permission returns 403 `userHasNoAcqUnitsPermission`; GET of a read-protected org by a non-member returns 403. (`ProtectionServiceImpl.java`; Karate `acquisitions-api-tests.feature`)
10. `Donor` organizations feed donor fields on funds and order lines; privileged donor info is separately permissioned. (docs.folio.org Finance › Donor information: a fund's donors are copied to the PO line when the fund is applied; Organizations › Privileged donor information)
11. **EDI `Schedule EDI` Time is shown in tenant timezone but stored as UTC** (`scheduleParameters.timeZone: "UTC"` in `/data-export-spring/configs/`); check both UI value and API value. (C965832; `utils.js` `getTenantTime`)
12. A scheduled export's real effect is a `Successful` job row in **Export Manager** (`Organizations` toggle, `Integration type` filter), not the saved config. (C965832, C1621423)
13. Integration requiredness is driven independently by File format (EDI vs CSV → EDI codes + account numbers) and Transmission method (FTP vs File download → Server address + FTP port); Ordering only locks both to EDI/FTP and shows Scheduling. (C688767; `utils.js`)
14. Duplicating an integration copies every field including FTP credentials, schedule and account numbers into `Copy of <name> (<date, time>)`; since UIORGS-481 both copies show their account numbers in Edit with the in-use warning. (`OrganizationIntegrationView.js`; UIORGS-441, UIORGS-481)
15. Privileged donor contacts are excluded from the Contact-people `Add contact` lookup; verified for a user with full privileged-donor rights, other roles not tested. (C423630, UIORGS-397)
16. Tenant setting `Enable banking information` off ⇒ Banking information accordion absent for all users regardless of permissions. (C423547)
17. Bank account types in use by any organization cannot be deleted — second popup `Cannot delete account type`. (C590820; UIORGS-435, MODORGSTOR-163)
18. Version History never shows Privileged donor information or Agreements, and shows each version's point-in-time accordion contents. (C663330)
19. `Add contact` / `Add donor` pickers page at 50; selection persists across pages; "Select all" = current page only. (C359169, C422163)
20. `Date created` facet uses the tenant timezone for From/To boundaries. (C466130)
21. Required organization fields: `Name`, `Code`, `Organization status` (`Active`/`Inactive`/`Pending`). (acq-models `organization.json`)
22. A new or changed code that starts or ends with whitespace is rejected with 422 `organizationCodeWhitespace`; an unchanged legacy code with whitespace still saves. (MODORG-99; `ProtectionServiceImpl.java` `validateCode`)
23. Banking information is saved only for vendors with the tenant setting on; it needs both `UI-Organizations` edit and the matching Banking-Information action. (`useBankingInformationManager.js`; C423518, C423513, C423514)
24. Keyword search `All` matches bank account number only for users with banking view permission. (`useBuildQuery.test.js`; C423426, C423427, C423432)
25. Deleting an organization with banking information deletes the banking records and the organization. (MODORGSTOR-172; C613152)
26. Privileged donor information accordion requires `Donor` checked plus `Privileged-Contacts - view`; it works for vendor and non-vendor orgs. (`OrganizationDetails.js`; C423618, C423690)
27. Only one default integration per organization: `Default integration` is disabled when another default exists; a default integration has no `Account numbers` field and never shows the in-use warning. (`IntegrationInfoForm.js`, `AccountNumbersInUseBanner.js`)
28. An account number used by another integration **of the same type** is not offered for selection; numbers already stored on the edited integration always stay visible and removable; cross-type reuse (Ordering vs Claiming) is allowed. (UIORGS-481; `utils.js` `buildAvailableAccounts`; C350552 for same type)
29. Changing Integration type to Claiming clears the saved EDI schedule; Ordering forces FTP + EDI and export type `EDIFACT_ORDERS_EXPORT`. (`IntegrationInfoForm.js`; UIORGS-442)
30. Field validation follows the current Integration type / Transmission method / File format immediately after a change. (UIORGS-468)
31. An Ordering integration must save as EDIFACT even if a token refresh happens while the form is open. (UIORGS-472; C825311)
32. Deprecated acquisition methods are hidden for new selection in `Automate order export for acquisition methods`; saved ones stay selected with `(deprecated)` and the export keeps working. (UIORGS-487; C1621423)
33. Scheduling validation: period must be chosen, frequency positive, Time required, Monthly `Day` 1–31. (`SchedulingForm.js`; C434063, C434064)
34. `Server address` must be a valid URL when Transmission method = FTP. (`utils.js` `validateFTPServerAddress`)
35. `EDI naming convention` cannot be set or edited (static token value). (C350758; `EdiForm.js`)
36. Integrations that existed before the type field (Ramsons → Sunflower upgrade) are migrated to `Ordering`; the first migration failed and was fixed in mod-organizations-storage 4.9.1. (UIORGS-442 closed 6.0.0; MODORGSTOR-181)
37. Version History does not list metadata fields as changes; metadata-only saves appear as "System maintenance update" cards. (UIORGS-482 + QA comment; C663330)
38. Number generator `Off` hides `Generate vendor code`; `On, field not editable` locks Code after generation; `On, field editable` keeps it editable. (C700855; UIORGS-337, UIORGS-462)
39. `Settings (Organizations): View settings` gives read-only settings; Banking information settings are hidden without banking view permission. (C407766: Categories/Types `New` inactive and Actions column empty; UIORGS-436)
40. Categories and organization types in use cannot be deleted (`This category cannot be deleted, as it is in use by one or more records.`). (en.json `settings.*` delete-guard strings; C367990 for the type guard step; C367989 covers only the successful delete of an unused category, not the guard)
41. Alternative names, vendor terms and accounts rows, once added, must be completed (Alias / Name / required account fields) or removed before save. (docs.folio.org; `OrganizationSummaryForm.js` Alias `validateRequired`)
42. Edit audit events for an organization carry both the original and the updated snapshot; create events only the new one. (MODORGSTOR-201; Karate `audit-event-organization.feature`)
43. Organizations are tenant-local in ECS; there is no share/central update for organizations yet. (MODORG-78..81 In Refinement; MODORGSTOR-197)

---

## Authoring style (measured 2026-07-23)

Organizations: **`Func` ~85%** / Other ~15%, median ~8 steps, `User Journey` ~15%. Cases are record-CRUD and accordion management (summary, contacts, interfaces, integrations/EDI, accounts, banking, privileged donor) plus Settings. Use `Functional`. Journey-flagged cases walk a multi-accordion setup or an integration→Export-Manager effect. Verify effects at their real destination. Preconditions carry the org record with the relevant accordion populated, the tenant kill-switch settings, and Eureka capability sets in `type - resource - action` form (see table above).

---

## Known Gaps / Items to Verify

- [ ] C688767 step 19 expects the duplicate's account number "NOT displayed" in Edit — outdated after UIORGS-481 (7.1.0): the stored number is now shown with the in-use warning. Case needs an update.
- [ ] C350552 / Cypress `verify-two-integrations-cannot-have-same-account-number.cy.js` cover same-type (Ordering) exclusion only; the new cross-type allowance (Ordering vs Claiming may share a number, UIORGS-481) has no case.
- [ ] UIORGS-487 (In QA) and UIORGS-485 (In Review) were not closed on 2026-11-24; behaviour is merged in master (CHANGELOG 7.1.0) — confirm in the Umbrellaleaf build.
- [ ] "System maintenance update" card text comes only from a UIORGS-482 QA comment; not in ui-organizations `en.json` (likely a shared stripes-acq-components string) — verify before asserting.
- [ ] `organizationCodeWhitespace` has no ui-organizations translation key: expected UI = `Organization could not be saved` plus a second callout with the backend message (`Organization code must not start or end with whitespace`) per `handleSaveErrorResponse.js` — verify; no TestRail or Karate case yet (MODORG-99).
- [ ] `Required!` vs `Required`: C688767 uses `Required!`, C434063 uses `Required` for the Day field — the text comes from stripes-acq-components, check in env.
- [ ] docs.folio.org says a vendor can be deleted even with linked orders/invoices; no TestRail case found for that path.
- [ ] Number generator settings capability `Settings - UI-Organizations Settings NumberGenerator - Manage` is confirmed by docs.folio.org and `package.json` only; no TestRail precondition uses it yet.
- [ ] C423502 Eureka precondition lists `Data - UI-Organizations Integrations Creds - Edit` although the case is about the `view` permission (legacy line says `…: view`) — likely a typo in the case; check before reusing.
- [ ] Rule 6: how Orders/Invoices use the vendor's expected receipt / invoice / activation / renewal / subscription intervals is not verified from a source.
- [ ] Rule 15 / C423630 verified only for a user holding `Privileged-Contacts - Edit` + `UI-Organizations - Edit`; behaviour for view-only users is not tested.
- [ ] Category delete guard (`This category cannot be deleted…`) has no TestRail step (C367989 covers only a successful delete); only en.json confirms the text.
- [ ] Integration details are stored in mod-data-export-spring, so whether integration add/remove shows in organization Version History (UIORGS-362, closed without fix version) is unconfirmed.
- [x] Eureka capability-set strings — now taken from TestRail preconditions and docs.folio.org (table above).
- [x] Interface flow labels — `Add interface` opens the find-interface lookup, `New` creates (en.json `interface.addInterface`, `interface.newInterface`).
- [x] "Save & keep editing" on Edit — covered by C656335 and UIORGS-449 (Create/Clone/Edit).

> Refresh log: 2026-07-22 N≥10 audit (14 cases from section 107); 2026-07-23 spot-check (C466130); 2026-11-07 delta (UIORGS-481, UIORGS-482, MODORGSTOR-201 — folded into the sections above on 2026-11-24); 2026-11-24 full refresh against GitHub master, Jira, TestRail (93 cases listed, 8 read in full; remediation 2026-11-25 read C423547, C423502, C423630, C407766, C367989, C367990 in full), Karate, Cypress, docs.folio.org.
