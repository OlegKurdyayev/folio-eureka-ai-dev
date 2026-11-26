---
area: open-access
generated_on: 2026-11-25
generator_version: manual (build-app-context method, refresh)
Release: Umbrellaleaf R2 2026
jira_status: ok
---

# FOLIO Open Access — Business Logic Context

> Built: 2026-11-07 · Refreshed: 2026-11-25 · Release: Umbrellaleaf R2 2026 (master: ui-oa 3.2.0 "IN PROGRESS", mod-oa 3.4.0 "IN PROGRESS"; last releases ui-oa 3.1.0 2026-07-08 and mod-oa 3.3.0 2026-07-06 = Trillium-compatible per UIOA-248 / MODOA-85) · Builder: build-app-context method, manual
> Sources: TestRail suite 21 section 115129 "Open Access" (Journals 115130, People 115131, Requests 115132, Publication charges 117721, Checklists 117722 — 12 Smoke cases, created 2026-06, full steps read for all 12 cases: C1373188, C1373195, C1373196, C1373197, C1373198, C1373199, C1348712, C1348713, C1348715, C1348717, C1348718, C1348719); GitHub folio-org/ui-oa (translations/ui-oa/en.json — en_US.json identical, 449 keys; package.json; CHANGELOG.md; src/), folio-org/mod-oa (service/src/main/okapi/ModuleDescriptor-template.json, grails-app/domain, HousekeepingService, NEWS.md), folio-org/app-oa (application descriptor); stripes-testing (cypress/e2e/fse/*/oa-*); folio-integration-tests (no OA project); Jira UIOA, MODOA, ERM (OA-related); Confluence "Open Access Requests" (page 5673362).
> Fix versions are not set on UIOA/MODOA issues — releases are tracked by release tasks (UIOA-248 = 3.1.0 Trillium, UIOA-244 = 3.0.0 Sunflower, UIOA-232 = 2.2.0 Ramsons) and CHANGELOG/NEWS.
> Tags: [source] = GitHub only (verify in env before strict assertion); [case Cxxx] = from TestRail case; (KEY) = Jira.

## What is Open Access

The **Open access** app (ui-oa + mod-oa, ERM family, `meta.title` = "Open access") manages requests to publish books and journal articles as open access, funded by direct charges (APCs/BPCs) or by transformative agreements (Confluence 5673362). Records: **Publication requests**, **People** (parties: corresponding authors / request contacts), **Journals** (works with title instances), **Charges** (with payers, discount, tax, exchange rate, links to Invoices app invoice/invoice line), **Correspondence**, **Checklists**, **Publication statuses**, **Funding**, linked **Agreement** (Agreements app). Settings hold pick lists, checklist item definitions, HRID prefix and default tax. OpenAPC reports can be run from the Actions menu.

Integrations (ui-oa package.json `optionalOkapiInterfaces`): `erm` 5.0/6.0/7.0 (Agreements), `invoice` 7.0, `organizations` 1.0, `finance.exchange-rate` 1.0; required `oa` 1.0/2.0. Request numbers come from Service interaction number generator `openAccess`, sequence `requestSequence` (ui-oa src/routes/PublicationRequestCreateRoute; UIOA-224).

## Record Hierarchy

```
Publication request (requestNumber, requestDate, requestStatus, closureReason, retrospectiveOA)
├── Corresponding author → RequestParty → Party (person)
├── Request contact      → RequestParty → Party (or "Use corresponding author")
├── Publication (publicationType, title, subtype, publisher, license, DOI, URL, authorNames, identifiers, localReference)
│   ├── Journal article → Work (journal) → Title instances (Print/Electronic) → Identifiers (ISSN/EZB/ZDB)
│   └── Book → bookDateOfPublication, bookPlaceOfPublication
├── Publication statuses[] (status, date, note)
├── Fundings[] (funder, aspectFunded)
├── External request IDs[]
├── Agreement (0..1, from Agreements app) | withoutAgreement flag
├── Correspondence[] (correspondent, date, mode, category, status, content)
├── Charges[] (category, status, amount+currency, exchangeRate, discount+type, tax, paymentPeriod, payers[], invoiceReference, invoiceLineItemReference)
└── Checklist items[] (definition from Settings; outcome, status Visible/Hidden, notes[])
```
(mod-oa grails-app/domain/org/olf/oa/*.groovy)

## UI structure (from en.json, routes and cases)

- Search & filter pane segments: Requests (default — C1373196), People, Journals (`searchAndFilter.requests/people/journals`).
- Routes (ui-oa src/index.js): `/oa/publicationRequests`, `/create`, `/:id/edit`, `/:id/charge/create`, `/:prId/charge/:chId`, `/:prId/charge/:chId/edit`, `/:prId/charge/:chId/linkInvoice`, `/:id/correspondence/create`, `/:prId/correspondence/:cId(/edit)`; `/oa/people`, `/oa/people/create`, `/oa/people/:id/edit`; `/oa/journals`, `/oa/journals/:id/edit` (journal creation is a modal from the Journals list).
- Request form accordions: Corresponding author, Request contact, Publication, Publication status, Funding, Agreement; all open by default (C1373188).
- Request view accordions: Corresponding author, Request contact, Publication, Publication status, Funding, Agreement, Correspondence, Charges — each header shows a count; Correspondence/Charges headers carry "Add correspondence"/"Add charge" (C1373195). Checklist pane opens from a bullet-list icon at top right of the view pane (C1373195, C1373199).
- Requests Actions menu: "New", "Run OpenAPC report" (C1373188).
- Settings > Open Access pages: Publication requests (HRID prefix, Default tax), Pick lists, Pick list values, Checklist items (ui-oa src/settings/index.js; en.json `settings.*`).

## API surface (mod-oa ModuleDescriptor-template.json, interface `oa`)

| Endpoint | Methods | permissionsRequired |
|---|---|---|
| `/oa/publicationRequest`, `/{id}` | GET / POST / PUT / DELETE | `oa.publicationRequest.collection.get`, `.item.get/.post/.put/.delete` |
| `/oa/party`, `/{id}` | GET / POST / PUT / DELETE | `oa.party.*` |
| `/oa/works`, `/{id}` ; `/oa/works/citation` | GET, PUT ; POST | `oa.works.collection.get/.item.get/.item.put` ; `oa.works.item.post` (no DELETE for works) |
| `/oa/titleInstances`, `/{id}` | GET | `oa.titleInstances.*.get` |
| `/oa/scholarlyWork`, `/{id}` | GET / POST / PUT / DELETE | `oa.scholarlyWork.*` |
| `/oa/charges`, `/{id}` | GET / POST / PUT / DELETE | `oa.charges.*` |
| `/oa/correspondence`, `/{id}` | GET / POST / PUT / DELETE | `oa.correspondence.*` |
| `/oa/checklistItems`, `/{id}` (item definitions) | GET / POST / PUT / DELETE | `oa.checklistItems.*` |
| `/oa/refdata`, `/oa/refdata/{domain}/{property}`, `/{id}` | GET / POST / PUT / DELETE | `oa.refdata.*` |
| `/oa/settings*` | GET / POST / PUT / DELETE | `oa.settings.get/post/put/delete` |
| `/oa/reports/{reportname}` | GET | `oa.reports.item.get` |
| `/oa/externalApi/*` | all | none |

Backend permission sets: `oa.<resource>.view` (collection+item get), `.edit` (= view + post + put), `.manage` (= edit + delete); `oa.settings.read/write`, `oa.refdata.read/write/manage`. No Karate project covers these endpoints (see Coverage).

## Permissions and Eureka capability sets

| UI permission (package.json) | Display name | Visible | Eureka capability set (as used in TestRail) |
|---|---|---|---|
| `ui-oa.all-open-access.view` | Open Access Requests: Search & view all | yes | (capability set name not confirmed — verify) |
| `ui-oa.all-open-access.edit` | Open Access Requests: Edit all | yes | (capability set name not confirmed — verify) |
| `ui-oa.all-open-access.manage` | Open Access Requests: Manage all | yes | (capability set name not confirmed — verify) |
| `ui-oa.allSettings.manage` | Settings (Open Access): Manage all Open Access settings | yes | (capability set name not confirmed — verify) |
| `ui-oa.publicationRequest.view/.edit/.manage` | Open Access Requests: Search & view / Edit / Manage publication requests | no | Data - UI-Oa Publication Request - View / Edit (C1373196, C1373188; TestRail spells lowercase `view`/`edit`; Cypress dictionary has no OA entries) |
| `ui-oa.party.view/.edit/.manage` | … people | no | Data - UI-Oa Party - View / Edit (C1373188, C1348717, C1348718; C1348719 precondition spells "Data - UI-Oa Parties - view" — see Known Gaps) |
| `ui-oa.journal.view/.edit` (no manage) | … journals | no | Data - UI-Oa Journal - View / Edit (C1373188, C1348712, C1348713, C1348715) |
| `ui-oa.charge.view/.edit/.manage` | View / Edit / Manage charges | no | Data - UI-Oa Charge - View / Edit (C1373195, C1373197) |
| `ui-oa.checklist.view` | Open Access Requests: View checklists | no | Data - UI-Oa Checklist - View (C1373199) |

Cross-app capability sets used in cases: Data - UI-Agreements Agreements - View (C1373195, C1373197), Data - UI-Invoice Invoice - Create (C1373198) (names as written in those TestRail preconditions; not in the Cypress dictionary check for OA, verify casing in env).
Eureka application: `app-oa` (folio-org/app-oa application.template.json) contains mod-oa + folio_oa, depends only on app-platform-minimal ^2.1.0 — Agreements/Invoices/Organizations are separate applications that must also be entitled for the cross-app flows.

## Status Values (mod-oa @Defaults; tenant-editable unless "internal")

| Field (refdata category) | Default values | Editable in Settings |
|---|---|---|
| Request status (`PublicationRequest.RequestStatus`) | Open, Closed | no (internal) |
| Reason for closure (`closureReason`) | Rejected | yes |
| Publication type | Journal Article, Book | no (internal) |
| Subtype / Publisher / License | Subtype 1 / Publisher 1 / License 1 (placeholders) | yes |
| Journal OA status (`Work.OaStatus`) | Gold, Hybrid | yes |
| In DOAJ (`Global.Yes_No`) | Yes, No | no |
| Institution level 1 (`Party.InstitutionLevel1`) | Faculty 1 | yes |
| Publication status | Submitted | yes |
| Funder / Aspect funded | Funder 1 / Research, Publication | yes |
| Charge category | APC, BPC | yes |
| Charge status (`Charge.ChargeStatus`) | Expected, Invoiced, Paid | no (internal) |
| Discount type | percentage, subtracted (UI: "%" / currency symbol) | no (internal) |
| Payer (`Payer.Payer`) | Library, DFG, Author | yes |
| Correspondence status | Awaiting Reply, Response Needed, Closed | yes |
| Correspondence mode / category | Email, Telephone / Invoice, Funding | yes |
| Request party role | Corresponding author, Request contact | no |
| Checklist item outcome / status | Yes, No, Other / Visible, Hidden | no |

Request status has no enforced transition rules in mod-oa; status changes are free edits (Open ↔ Closed).

## Settings

- Settings > Open Access > Publication requests: `hrid_prefix` ("HRID prefix") and `default_tax` ("Default tax"), section `PublicationRequests`, created by mod-oa HousekeepingService (UIOA-254 fixed them not displaying).
- Pick lists / Pick list values: tenant-editable refdata (non-internal categories above).
- Checklist items: definitions with Name (slug), Label, Description, Weight ("Item order weight, higher value means lower placement" — `settings.checklistItemDefinitions.weightPopover`).
- Charge currency = tenant primary currency (Settings > Tenant > Language and localization — C1373197).

## Existing cases (all Smoke, created 2026-06 per UIOA-256)

- Journals: C1348712 Create Journal; C1348713 Search or filter for journals; C1348715 Display journal
- People: C1348717 Create Person; C1348718 Search or filter for people; C1348719 Display person
- Requests: C1373188 Create an article request; C1373195 Request accordion display; C1373196 Search or filter for a request
- Publication charges: C1373197 Add a publication charge; C1373198 Create an invoice for a charge
- Checklists: C1373199 Complete a checklist item for a request
- Planned but not created (UIOA-257, Draft): OpenAPC report, Book request, Add publication status, Add correspondence, Add funding, Splitting charge by payer, Creating checklist items, Hiding checklist items, Inline creation of journal/person, Institutional hierarchy.

## Key Business Rules for Test Cases

### Publication requests
1. Creating a publication request shows the callout "Publication request {requestNumber} successfully created" (`publicationRequest.success.callout`, ui-oa translations/ui-oa/en.json) [source].
2. Request date and Status are required on the request form (ui-oa RequestInfoForm.js `requiredValidator`; C1373188).
3. "Reason for closure" is disabled unless Status = Closed (RequestInfoForm.js `disabled={requestStatus !== 'closed'}`; C1373188 "greyed out/inactive"); default closure reason value is "Rejected" (mod-oa PublicationRequest.groovy).
4. Request number is taken from number generator `openAccess`/`requestSequence` on submit (UIOA-224); if none is sent, mod-oa generates `hrid_prefix` + next value of `pubreq_hrid_seq` (PublicationRequest.groovy `generateHrid`).
5. If request status is null on save, mod-oa sets it to "New" (PublicationRequest.groovy `beforeValidate`) — reachable via API only, since the UI requires Status.
6. In the Request contact accordion "Use corresponding author" is checked by default and copies the corresponding author (C1373188).
7. Selecting Publication type "Journal article" shows the Journal details sub-section with a journal typedown (C1373188); saving without selecting a journal must not render an empty journal card (UIOA-239).
8. Journal typedown searches title and identifier values (ISSN) and offers "Exact title match" (`journal.exactTitleMatch`; UIOA-210, UIOA-213; PublicationJournal.js searchKey).
9. Adding a person to a request searches givenNames, familyName, mainEmail, alternateEmails.email and orcidId (UIOA-252; PartyTypedownForm.js).
10. Publication title max length 4096 (UIOA-212, MODOA-46); Publication URL must include http/https ("Please enter a valid URL (including 'http' or 'https')" — `publicationRequest.validate.publicationURL`).
11. Book date of publication accepts YYYY, YYYY-MM or YYYY-MM-DD only ("Please enter a date in the format YYYY, YYYY-MM or YYYY-MM-DD" — `publicationRequest.publicationYear.validate`; mod-oa regex on `bookDateOfPublication`).
12. Agreement accordion: checking "Request has no agreement" hides "Link agreement"; the view then reads "This request is not covered by an agreement" (C1373188; `publicationRequest.noAgreement`, `publicationRequest.agreementNotLinked`).
13. Linked agreement shows Name (link to Agreements app), Start date, End date, Status (C1373195). If Agreements is unreachable the form shows `publicationRequest.agreementsNotWorking` (UIOA-236 — was an environment/version issue).
14. "Retrospective open access" is a checkbox on the request (UIOA-187; `publicationRequest.retrospectiveOpenAccess`).
15. Deleting a request cascades its charges, publication statuses, fundings, identifiers, external IDs and agreement link (PublicationRequest.groovy `cascade: 'all-delete-orphan'`).

### Search & filter
16. Request search box covers publication title, request number, corresponding author and request contact full name, external request ID, DOI, local reference, publication identifiers (PublicationRequestsRoute.js searchKey; C1373196).
17. Request filters: Status, Charge status, Publication type, OA status, Publisher, Charge Payers, Institution level 1, Retrospective OA, Correspondence status, Request date range, plus checklist filter builder (PublicationRequestsRoute.js filterKeys; UIOA-164, UIOA-195; en.json `checklistFilter.*`).
18. Selected filter checkboxes must display as selected after the filtered search (UIOA-260 — regression candidate).
19. Request results: 25 per page with Previous/Next; default sort by request date descending; Request number column sortable asc/desc (C1373196). Count label "{number} records found" (`publicationRequests.recordsFound`).
20. People search covers mainEmail, givenNames, familyName, orcidId (C1348718: searching a word from each of family name, given name, ORCiD and main email must return the known person); filter Institution level 1; default sort familyName, givenNames (PartiesRoute.js). Count label "Found {total} people" (`parties.found#People`; UIOA-182).
21. Journal search covers title and identifier values; filters OA status, In DOAJ; only Title is sortable — ISSN columns are not (JournalsRoute.js; UIOA-238; C1348713). Same Search & filter pattern as requests/people: "%" returns all, 25 per page with Previous/Next "1-25"/"26-50", "Reset all" clears search box and filters and leaves an empty results pane (C1348713, C1348718; TestRail shared step 39).
22. Saving a request/person/journal/correspondence must not reset the current search and filters (UIOA-197).
23. Known open defect: names starting with an uppercase umlaut (e.g. "Özdemir") may not be found on DBs with C collation (UIOA-253, Draft) — do not assert as pass/fail without checking the env collation.

### People and journals
24. Person ORCID iD and main email must each be unique; duplicates fail with `party.uniqueOrcIdError` / `party.uniqueMainError` (mod-oa Party.groovy `unique: true`; MODOA-17, MODOA-19).
25. Person full name is built as "{givenNames} {familyName}" on save; lists show "{familyName}, {givenNames}" (Party.groovy; `party.familyNameOrdered`; C1373196).
26. Main and other email fields accept up to 255 characters (UIOA-205/206, MODOA-44); an added "Email address" field is required (C1348717).
27. Journal instance: EZB namespace only valid for Subtype "Electronic" (`journal.validate.notElectronicSubtype`); an ID+namespace combination must be unique across instances (`journal.validate.duplicateNamespaceID`; MODOA-35).
27a. Journal view pane: heading = journal title; shows title, In DOAJ, OA status and each instance with material type and identifier (C1348715; UIOA-73, UIOA-104, UIOA-119). Person view pane: heading "{family name}, {first name}", Name property "{title} {given name} {family name}", ORCiD ID, phone and mobile, Institution level 1 and 2, main email, other emails as bullet list, full street address (C1348719; UIOA-40/41/42/67/92/95).
28. Journals cannot be deleted — there is no `ui-oa.journal.manage` and no DELETE on `/oa/works` (UIOA-229; ModuleDescriptor).

### Charges
29. Charge form defaults: Charge currency = tenant primary currency, Exchange rate inactive and = 1 for the same currency, Tax (%) = Settings "Default tax" (C1373197; UIOA-243).
30. If no primary currency is set, charge creation is blocked with `publicationRequest.noCurrencySet` (en.json; trigger condition not checked in src/ — unverified) [source].
31. Amount to pay = (Net − Net×discount/100) × (1 + tax/100) for "%" and (Net − discount) × (1 + tax/100) for the currency-amount discount (mod-oa Charge.groovy `getEstimatedInvoicePrice`; UIOA-250 fixed the UI formula). Example: Net 110, Tax 10 → 121.00; +10% → 108.90; 10 currency → 110.00 (C1373197).
32. Changing Discount type on edit must be saved and recalculated (UIOA-251 — regression candidate).
33. Number fields (Net amount, Exchange rate, Discount, Tax %) accept a zero after the decimal point, e.g. 1.05 (UIOA-258); max 10 digits either side ("Maximum 10 digits either side" — `charge.validate.maxDigits`); negative values rejected ("Cannot be negative"/"Cannot be less than zero").
34. Decimals must persist with non-English locales (e.g. German) for charge and payer amounts (UIOA-222, UIOA-230).
35. Payment period must be a 4-digit year ("Must be a 4 digit number" — `charge.validate.fourDigits`; Charge.groovy regex `^([0-9]{4})?$`).
36. Payers: each has Payer, note, amount; the sum of payer amounts must not exceed the charge total ("The amount(s) should be equal to or less than the charge total amount" — `charge.payers.amountsMoreThanChargeTotal`); the remaining amount is pre-filled and must be rounded to 2 decimals (UIOA-215; MODOA-25).
37. Charge view header reads "Charge: request {requestNumber}" (`charge.publicationRequestCharge`; C1373198); Charges MCL in the request shows Description, Net amount, Calculated amount, Invoice line (C1373195).
38. Delete charge confirmation: "This charge will be permanently deleted." (`charge.deleteChargeMessage`, en.json).

### Invoices link
39. Action "Link invoice line" opens "Link invoice line to charge"; an invoice can be selected or created ("New Invoice" — Invoice number, Invoice date, Vendor organisation, Payment method, Batch group, Currency, "Use current exchange rate"/"Use charge exchange rate") (C1373198).
40. "New invoice line" asks for confirmation ("Select 'Submit' to go ahead."); the invoice line is created with status Open in the Invoices app (`charge.invoiceLine.newInvoiceLineConfirmMessage`, `charge.invoiceLine.status.open`; C1373198) — verify the invoice/line in the Invoices app, not only in OA.
41. New invoice lines cannot be generated for approved, cancelled or paid invoices (`charge.invoiceLine.noNewInvoiceLine`, en.json; enforcement logic not checked — unverified) [source].
42. If charge and invoice currencies differ, a "Conflicting currencies" warning shows and the line uses the invoice currency (`charge.invoiceLine.currencyDifferenceMessage`, en.json; unverified beyond the message text) [source].
43. Unlinking removes only the link — it does not delete the invoice or invoice line (`charge.invoice.unlinkInvoiceLineMessage`, `charge.invoice.invoiceHelperMessage`, en.json; UI buttons "Link/Unlink invoice line" in ChargeView.js).
44. Tax is not included in the created invoice line amount (UIOA-193, open documentation task) — check before asserting the invoice line total.

### Correspondence and checklists
45. Correspondence requires Correspondent, Date, Mode, Status and content (mod-oa Correspondence.groovy `nullable: false`); delete confirmation "This correspondence will be permanently deleted." (`correspondence.deleteCorrespondenceMessage`, en.json).
46. Checklist items are ordered by weight (higher sinks), then label (C1373199; `weightPopover`).
47. Checklist outcomes: Not set, Yes, No, Other; setting an outcome changes "Created: …" to "Updated: {date} {time}" (C1373199; `checklist.updated`).
48. Notes per checklist item: Save/Cancel active only after text entry; the counter increments (C1373199).
49. Hiding an item moves it to a "Hidden" accordion with a count; showing it again removes the accordion when empty (C1373199).
50. With no checklist item definitions the pane shows "The checklist can be used to manage individual publication requests" + link to Settings (`checklist.emptyMessage*`, en.json) [source].
51. Checklist item definition Name is a unique normalized slug (diacritics removed, special chars → "_"); Label may contain special/national characters (ChecklistItemDefinition.groovy; UIOA-211).

### Permissions (UI)
52. Charge "Edit" and "Link/Unlink invoice line" need `ui-oa.charge.edit`; charge "Delete" needs `ui-oa.charge.manage` (ui-oa ChargeView.js hasPerm); correspondence "Edit" needs `ui-oa.publicationRequest.edit` and "Delete" needs `ui-oa.publicationRequest.manage` (CorrespondenceView.js); an Actions menu with no allowed entries is not shown (UIOA-242).
53. "New"/Edit on requests, people, journals are wrapped in `ui-oa.publicationRequest.edit`, `ui-oa.party.edit`, `ui-oa.journal.edit` checks (ui-oa PublicationRequestsRoute.js, PartiesRoute.js, JournalsRoute.js `IfPermission`; also "Add charge"/"Add correspondence" in Charges.js/Correspondence.js and checklist edit/notes controls in Checklist/*.js need `ui-oa.publicationRequest.edit`).
54. Every user of the app needs settings read (default tax / HRID) via `module.oa.enabled` (UIOA-229, UIOA-249).

### Reports
55. OpenAPC report formats: OpenAPC APC, OpenAPC BPC, OpenAPC Transformative Agreement; parameters Institution, Payment period, Charge categories, Charge statuses; success callout `report.reportCreated` and file download (en.json `report.*`; MODOA-28/29/39).

### API edge cases
56. `POST /oa/charges` without `exchangeRate` returns 500 instead of 400/422 (MODOA-101, Open) — known defect, mark as such in API cases.

## Coverage

- Karate (folio-integration-tests): none — no mod-oa project (checked repo tree 2026-11-24).
- Cypress (stripes-testing): `cypress/e2e/fse/ui/oa-ui.cy.js` (TC196024 — app opens, "New publication request" pane), `cypress/e2e/fse/api/oa-api.cy.js` (TC196025 — GET publication requests 200); fragments `cypress/support/fragments/oa/oaSearchAndFilter.js`, `oaNewPublicationRequest.js`. FSE smoke only; no functional e2e.

## Known Gaps / Items to Verify

- [ ] Eureka capability sets for the visible "all" sets (`ui-oa.all-open-access.*`, `ui-oa.allSettings.manage`) — not used in any case; names inferred.
- [ ] `ui-oa.charge.edit` displayName is "Open Access Requests: Edit people" in package.json (copy-paste defect present since UIOA-233) — the capability label in the env may show that text.
- [ ] Release membership: OA is not in platform-complete package.json and Confluence 5673362 says it is not part of a flower release, while app-oa declares `"platform": "complete"` — confirm whether app-oa is installed on the target Umbrellaleaf env.
- [ ] C1373188 step 4 says "Save and close" becomes active once request date OR status is set, but both fields are required — verify actual button behaviour.
- [ ] `publicationRequest.noCurrencySet` points to "Settings > Open Access > Language and localization > Primary currency", but C1373197 uses Settings > Tenant > Language and localization — UI text likely wrong; assert the text verbatim but configure the tenant setting.
- [ ] mod-oa sets status "New" when status is null, but "New" is not among the request status defaults (Open, Closed) — API-only behaviour, confirm.
- [ ] UIOA-215 QA note: second payer pre-filled amount may still show float artefacts — verify rounding.
- [ ] UIOA-258 note: German locale view mode shows "." instead of "," as decimal separator.
- [ ] Request/charge pick lists are tenant-configurable — values beyond defaults are env-specific.
- [ ] No negative, permission-boundary, correspondence, book-request, OpenAPC or settings cases exist yet (UIOA-257 Draft).
- [x] C1348713, C1348715, C1348718, C1348719 — steps read 2026-11-25 (rules added; see rules 20, 21, 27a). All 12 Smoke cases are now read in full.
- [ ] C1348719 precondition spells the capability set "Data - UI-Oa Parties - view" while C1348718/C1373188/C1348717 use "Data - UI-Oa Party - view" — confirm the real name in the env.
- [ ] Capability set casing: TestRail OA cases spell `view`/`edit` lowercase; the file shows `View`/`Edit` (house style) — confirm exact casing in the env.

---

## Exact UI Texts

> From `folio-org/ui-oa` `translations/ui-oa/en.json` at HEAD (2026-11-24; en_US.json identical). `{x}` are placeholders; `<b>`/`<strong>` render bold. Use verbatim.

### Toast Messages

| Key | Text | Source |
|---|---|---|
| `refdata.deleteRefdataValue.confirmMessage` | <strong>{name}</strong> will be deleted. | [source] |
| `refdata.deleteRefdataValue.errorMessage` | <strong>Error: {label}</strong> was not deleted. {error} | [source] |
| `settings.checklistItemDefinitions.deleteDefinition.confirmMessage` | Checklist item <strong>{label}</strong> will be <strong>deleted</strong> | [source] |
| `settings.checklistItemDefinitions.deleteDefinition.errorMessage` | <strong>Error: {label}</strong> was not deleted. {error} | [source] |
| `party.uniqueOrcIdError` | <strong>Error:</strong> Party was not saved. OrcId must be unique. A Party with OrcId {duplicateElement} already exists | [source] |
| `party.uniqueMainError` | <strong>Error:</strong> Party was not saved. Main email must be unique. A Party with the main email {duplicateElement} already exists | [source] |
| `party.creationSuccessCallout` | <strong>Party created:</strong> {createdParty} | [source] |
| `party.updatedSuccessCallout` | <strong>Party updated:</strong> {updatedParty} | [source] |
| `journal.creationError` | <strong>Error:</strong> Unable to create a new journal, please check the details and try again | [source] |
| `journal.creationSuccess` | <strong>Journal created:</strong> {journalTitle} | [source] |
| `publicationRequest.success.callout` | Publication request {requestNumber} successfully created | [source] |
| `report.reportCreated` | <strong>Report created:</strong> {fileName}, the file will automatically download. | [source] |
| `charge.invoiceLine.newInvoiceLineConfirmMessage` | Select 'Submit' to go ahead. | [source] |

### Modal / Pane Titles

| Key | Text |
|---|---|
| `publicationRequest.createPublicationRequest` | New publication request (also asserted in stripes-testing oaNewPublicationRequest.js) |
| `publicationRequest.editPublicationRequest` | Edit publication request - {id} |
| `publicationRequest.requestTitle` | Request number - {number} |
| `charge.newCharge` / `charge.editCharge` | New charge / Edit charge |
| `charge.publicationRequestCharge` | Charge: request {requestNumber} |
| `charge.deleteCharge` | Delete charge |
| `charge.invoice.linkInvoiceLineToCharge` | Link invoice line to charge |
| `charge.invoiceLine.newInvoiceLineMessage` | A new invoice line will be added to invoice {invoiceNo} with the below details: |
| `correspondence.newCorrespondence` / `editCorrespondence` / `deleteCorrespondence` | New correspondence / Edit correspondence / Delete correspondence |
| `party.newPerson` / `party.editPerson` | New person / Edit person |
| `journal.newJournal` / `journal.editJournal` | New journal / Edit - {title} |
| `settings.checklistItemDefinitions.newChecklistItem` / `deleteHeading` | New checklist item / Delete checklist item |
| `report.runOpenApcReport` | Run OpenAPC report |

### Validation / Warning Messages

| Key | Text |
|---|---|
| `charge.validate.notLessThanZero` | Cannot be less than zero |
| `charge.validate.notNegative` | Cannot be negative |
| `charge.validate.maxDigits` | Maximum 10 digits either side |
| `charge.validate.fourDigits` | Must be a 4 digit number |
| `charge.payers.amountsMoreThanChargeTotal` | The amount(s) should be equal to or less than the charge total amount |
| `charge.payers.remainingAmount` | <strong>Remaining amount to be split between payers: {amount}</strong> |
| `charge.updateExchangeRate.disabledToolTip` | Exchange rate for {exchangeCurrency} is not known, Please enter manually |
| `charge.invoiceLine.noNewInvoiceLine` | You cannot generate new invoice lines for invoices which are approved, cancelled or paid |
| `charge.invoiceLine.currencyDifferenceMessage` | Warning: <strong>Conflicting currencies</strong>. The currency on the request charge is {chargeCurrency}. This invoice line will be created using the currency on the invoice: {invoiceCurrency} |
| `charge.invoice.unlinkInvoiceLineMessage` | The associated invoice and invoice line will be unlinked from this charge. |
| `publicationRequest.noCurrencySet` | Unable to create charge because the currency is unknown. Please set a currency in Settings > Open Access > Language and localization > Primary currency (note leading space in en.json) |
| `publicationRequest.agreementsNotWorking` | <strong>Error :</strong> Agreements is currently unavailable, please try again later or contact your system administrator |
| `publicationRequest.validate.publicationURL` | Please enter a valid URL (including 'http' or 'https') |
| `publicationRequest.publicationYear.validate` | Please enter a date in the format YYYY, YYYY-MM or YYYY-MM-DD |
| `journal.validate.notElectronicSubtype` | EZB is not a valid option for the subtype 'Print'. Please rectify |
| `journal.validate.duplicateNamespaceID` | This ID and namespace combination exists in another instance, please rectify |
| `settings.checklistItemDefinitions.validateNoSpecial` | Cannot include special characters |

### Empty-state Texts

`publicationRequest.emptyFundings` "No funding for this request"; `publicationRequest.emptyPublicationStatuses` "No publication statuses for this request"; `publicationRequest.emptyCorrespondence` "No correspondence for this request"; `publicationRequest.emptyCharges` "No charges for this request"; `charge.payers.isEmptyMessage` "There are no payers associated with this charge"; `charge.invoiceNotLinked` "No invoice linked to this charge"; `charge.invoiceLineNotLinked` "Invoice line not linked"; `journal.noTitleInstances` "No title instances for this journal"; `checklist.emptyMessage` "The checklist can be used to manage individual publication requests".

### Key Labels

Request: "Request number", "Request date", "Status", "Reason for closure", "Retrospective open access", "Corresponding author", "Request contact", "Use corresponding author", "Publication type", "Publication title", "Add publication status", "Add funding", "Funder", "Aspect funded", "Request has no agreement", "Show checklist". Charge: "Category", "Status", "Charge description", "Amount to pay, with tax and discount applied", "Discount type", "Discount note", "Tax (%)", "Payment period", "Payers", "Add payer", "Payer note", "Amount in charge currency", "Charge breakdown", "Link invoice line", "Unlink invoice line", "New Invoice", "New invoice line". Checklist outcomes: "Not set", "Yes", "No", "Other". (en.json `publicationRequest.*`, `charge.*`, `checklist.outcome.*`.)
