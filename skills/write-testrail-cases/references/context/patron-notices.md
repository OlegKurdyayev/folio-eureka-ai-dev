# FOLIO Patron Notices — Business Logic Context

> Built: 2026-07-21 · Refreshed: 2026-11-25 · Release: Umbrellaleaf R2 2026 (master: ui-circulation 13.0.0 in progress, mod-circulation 24.6.0-SNAPSHOT, mod-circulation-storage 17.6.0-SNAPSHOT, mod-template-engine 1.24.0-SNAPSHOT) · Builder: build-app-context (refresh mode, full merge of the 2026-11-07 delta)

> Sources (refresh 2026-11-24): GitHub folio-org/ui-circulation master (`translations/ui-circulation/en.json`, `src/settings/PatronNotices/**`, `src/settings/NoticePolicy/**`, `src/settings/Validation/**`, `src/constants.js`, `package.json`, CHANGELOG), mod-circulation-storage (`descriptors/ModuleDescriptor-template.json`, `ramls/patron-notice-policy.json`, `ramls/scheduled-notice.json`, `PatronNoticePoliciesAPI.java`), mod-template-engine (ModuleDescriptor, NEWS.md, `TemplateContextPreProcessor.java`, `TemplateServiceImpl.java`), mod-circulation (ModuleDescriptor timers, `domain/notice/*`, NEWS.md), mod-notify (ModuleDescriptor); folio-integration-tests; stripes-testing; docs.folio.org source (`content/en/docs/Settings/Settings_circulation/settings_circulation.md`); Confluence FOLIOtips "Settings - Circulation - Patron notice templates" (page 5672543, v. 2026-07-30); Jira UICIRC / CIRC / MODTEMPENG / MODNOTIFY / CIRCSTORE / UXPROD Done-Closed 2024-06..2026-11 (~80 issues listed, 33 read in detail); TestRail suite 21 sections 331, 17059, 17139, 17145, 111082 (79 cases listed, key ones read).
> TestRail: suite 21, parent section **328 "Patron notices"** with children **331 "Patron notices settings"** (template and policy editor cases), **17059 "Loan patron notices"**, **17139 "Request notice triggers"**, **17145 "Fees/fines patron notices"**, **111082 "ECS patron notices"**. Settings live under `Settings > Circulation > Patron notices` (`settings.index.patronNoticesSettings`): **Patron notice templates** (message content), **Patron notice policies** (which template fires on which event and when), **TLR patron notice templates** (`settings.index.tlrPatronNotices`). The runtime effect (a notice in the mailbox) is asserted in the circulation apps — see check-in.md / requests.md / loans.md.
> UI strings: `en.json` is the source of truth. `en_US.json` (Lokalise copy) lags: it still lacks the 11 Text-message/reorder keys below and still carries the removed key `settings.patronNotices.emailOrPrint` ("Email or print").

---

## What is Patron Notices

Patron notices are automated messages sent to patrons/requesters on circulation events. A **template** defines the content (Notice format, Subject for email, Body) with **tokens**; a **notice policy** binds templates to **triggering events** with **timing** and **frequency**; a circulation rule assigns a notice policy to a patron-group × material-type × loan-type × location combination. A real notice requires: an active template + a notice policy referencing it for the event + a circulation rule pointing at that policy + the triggering event occurring. Delivery: mod-circulation builds the context and calls mod-notify `POST /patron-notice`, which renders through mod-template-engine and hands off to the sender.

---

## Patron notice templates [confirmed]

`Settings > Circulation > Patron notice templates`. New pane title `New patron notice template` (`settings.patronNotices.newLabel`); existing template pane title = template name.

**Accordions** (`PatronNoticeForm.js`): `General information` (metadata + name/Active/Description/Category) and `Patron notice content` (`settings.patronNotices.patronNoticeContent`, UICIRC-1350). The old `Email` accordion with the `Print only` checkbox is gone; `Email or print` was removed — do not assert it.

**General information fields:** `Patron notice template name` (required, unique), `Active`, `Description`, `Category` (required).

**Patron notice content fields** (`PatronNoticeEmailSection.js`, UICIRC-1350):
- `Notice format` (required Select, InfoPopover `settings.patronNotices.noticeFormat.infoPopover`). Options in order: `Select notice format` (empty default), `Email`, `Print only`, `Text message`.
- `Subject` — enabled and required **only** for `Email`; disabled for empty/`Print only`/`Text message`. InfoPopover: `To enable the Subject field, select Email in the Notice format drop-down list.` (`settings.patronNotices.subject.infoPopover`, bold markup on Subject/Email/Notice format).
- `Body` — disabled until a Notice format is selected, then required. InfoPopover: `To enable the Body field, select an option in the Notice format drop-down list.` (`settings.patronNotices.body.infoPopover`). For `Text message` the editor is plain text and image tokens (`item.barcodeImage`, `item.instanceHridImage`, `user.barcodeImage`) are disabled in `Add token`.
- Text-message InfoPopover text: `If text message is selected, it has a maximum limit of 160 characters, including spaces, numbers, and symbols. If more than 160 characters are used, multiple text messages will be sent.` No character counter or validation is enforced in UI (UXPROD-1532 comment 2026-05-04: token values are resolved at send time).
- Switching Notice format back to a previously selected one restores the Subject/Body typed for that format; first switch to Email restores the saved template values; other not-yet-visited formats start blank (`resolveNoticeFormatFieldValues.js`).
- Saved payload: format stored as `additionalProperties.noticeFormat` (`email` / `print` / `textMessage`), legacy `additionalProperties.printOnly` = true only for Print; `outputFormats` = `text/plain` for Text message, else `text/html`; for non-email formats the BE-required `header` is auto-filled with `print` / `textMessage` (`PatronNotices.js` handleBeforeSave). Legacy templates without `noticeFormat` open as Print if `printOnly` = true, else Email (`getNoticeFormat.js`).
- Buttons: `Save` / `Save new` labels exist (`settings.patronNotices.saveLabel`, `saveNewLabel`); footer Save is disabled while the form is pristine or submitting (`PatronNoticeForm.js`). Preview modal: `Preview of patron notice template` (form) / `Preview of patron notice template - {name}` (view).

**Categories** (`settings.patronNotices.categories.*`, order in dropdown): `Loan`, `Request`, `Automated fee/fine charge`, `Automated fee/fine adjustment (refund or cancel)`, `Manual fee/fine charge`, `Manual fee/fine action (pay, waive, refund, transfer or cancel/error)`.

**Token groups** (`Add token`): `Item`, `Effective location`, `Loan`, `Request`, `User`, `User Address`, `Fee/fine charge`, `Fee/fine action`; loops `Multiple loans` (`{{#loans}}…{{/loans}}`) and `Multiple fee/fine charges` (`{{#feeCharges}}…{{/feeCharges}}`, enabled only for category `Automated fee/fine charge` — `TokensList.js` isDisabledLoop). Keys `Multiple requests` / `Multiple fee/fine actions` exist in en.json but are not wired in TokensList.js.

**Token availability by category** (`src/settings/PatronNotices/tokens.js`):
- Item / Effective location / User / User Address tokens: all categories. Item tokens in order: `item.title`, `item.titleShort`, `item.primaryContributor`, `item.allContributors`, `item.barcode`, `item.barcodeImage`, `item.callNumber`, `item.callNumberPrefix`, `item.callNumberSuffix`, `item.displaySummary`, `item.enumeration`, `item.volume`, `item.chronology`, `item.yearCaption`, `item.materialType`, `item.copy`, `item.numberOfPieces`, `item.descriptionOfPieces`, `item.loanType`, `item.accessionNumber`, `item.administrativeNotes`, `item.datesOfPublication`, `item.editions`, `item.physicalDescriptions`, `item.instanceHrid`, `item.instanceHridImage`, then `item.effectiveLocation{DiscoveryDisplayName,Specific,Library,Campus,Institution}`.
- User: `user.firstName`, `user.lastName`, `user.middleName`, `user.preferredFirstName`, `user.barcode`, `user.barcodeImage`, `user.primaryDeliveryAddressType`, `user.primaryAddressLine1/2`, `user.primaryCity`, `user.primaryStateProvRegion`, `user.primaryZipPostalCode`, `user.primaryCountry`.
- Request (category `Request` only): `request.servicePointPickup`, `request.requestExpirationDate(Time)`, `request.holdShelfExpirationDate(Time)`, `request.reasonForCancellation`, `request.additionalInfo`.
- `loan.additionalInfo`: `Loan` only. Other loan tokens (`loan.dueDate(Time)`, `loan.initialBorrowDate(Time)`, `loan.checkedInDate(Time)`, `loan.numberOfRenewalsAllowed/Taken/Remaining`): Loan + Automated fee/fine charge/adjustment + Manual fee/fine action.
- `feeCharge.*` (`owner`, `type`, `paymentStatus`, `chargeDate(Time)`, `amount`, `remainingAmount`, `additionalInfo`): all four fee/fine categories. `feeAction.*` (`type`, `actionDate(Time)`, `amount`, `remainingAmount`, `additionalInfo`): Manual fee/fine action + Automated fee/fine adjustment.

**Item-group tokens confirmed live (C692204, story CIRC-2260/UICIRC-1178):** `{{item.accessionNumber}}`, `{{item.administrativeNotes}}`, `{{item.datesOfPublication}}`, `{{item.editions}}`, `{{item.physicalDescriptions}}`, `{{item.instanceHrid}}`, `{{item.instanceHridImage}}`. Selecting a token in `Add token` inserts it into the Body; `Preview` shows hard-coded sample values (same for every template, multiples not expanded — Confluence FOLIOtips 5672543).

**`item.titleShort` (UICIRC-1353, C1453636, C1453637):** listed below `item.title` and above `item.primaryContributor`; preview shows the first 25 characters (preview sample `The Wines of Italy: A Comprehensive Guide to Regional Varietals` → `The Wines of Italy: A Com`). The backend generates `item.shortTitle`, not `item.titleShort`; MODTEMPENG-141 (Open, created 2026-11-25) confirms `{{item.titleShort}}` renders empty in real notices — see Known Gaps.

**Errors / guards:** name `A patron notice with this name already exists` (`settings.patronNotices.errors.nameExists`); delete guard `Cannot delete Patron notice template` / `This Patron notice template cannot be deleted, as it is in use by one or more records.` (UI checks whether any notice policy references the template id — `PatronNotices.js` isTemplateExist; BE also refuses with InUseTemplateException — `TemplateServiceImpl.java`); delete confirm `Confirm delete` / `Are you sure you want to delete this patron notice?`; predefined: `This is a predefined notice and cannot be deleted`; Subject `Should not begin with a space` (`settings.validate.subject`, C356783 / UICIRC-782).

**Record metadata dropdown [confirmed — C411862, C380408, C387439, C387440]:** the template's Edit pane shows a `Record last updated: {Month/Day/Year time}` dropdown (labeled `Record created` on first save). Expanding it reveals `Record created` (frozen) with its `Source` (creator name, or `Unknown user` if migrated) and `Record last updated` with its `Source` (last editor). Editing ANY field and clicking `Save & close` updates `Record last updated` and its `Source`; `Record created` never changes. Viewing it needs `users.collection.get` (added to view-templates set in UICIRC-1232; missing it caused 403 "updaters" popups).

---

## Patron notice policies [confirmed]

`Settings > Circulation > Patron notice policies`. Pane `Patron notice policies`; new `New patron notice policy`, edit `Edit patron notice policy`. Accordions: `General information`, `Loan notices - sent to borrower`, `Fee/fine notices`, `Request notices - sent to requester`. Fields: `Patron notice policy name` (required, unique — BE 422 `'<name>' name in not unique`, `PatronNoticePoliciesAPI.java`), `Description`, `Active`.

**Notice card** (`NoticeCard.js`): header = drag handle (tooltip `Drag & drop`) + `Notice {counter}`, then per-card icons plus `Add notice above` and trash `Delete notice` (tooltips and aria-labels). Fields: `Template` (required) `via` `Format` (required) + `Triggering event` (required). No default is pre-selected in Template/Format/Triggering event (UICIRC-1349). For time-based events the `Send` select appears; for `Before`/`After` it adds `by` + duration + interval, and `Frequency` (`One Time` / `Recurring`); Recurring adds `and every` + duration + interval. Interval options: `Minute(s)`, `Hour(s)`, `Day(s)`, `Week(s)`, `Month(s)`, `Year(s)`.

**Format dropdown:** `Email`, `Text message` (stored `Email` / `SMS`; UICIRC-1349, `noticesFormats` in `src/constants.js`). BE schema also accepts `Print`, but the UI does not offer it.

**Template dropdown:** lists only **active** templates of the matching category — Loan notices → `Loan`; Request notices → `Request`; Fee/fine notices → `Automated fee/fine charge` + `Automated fee/fine adjustment (refund or cancel)` (`NoticePolicyForm.js` + `get-templates.js`). Manual fee/fine templates never appear in a policy (they are chosen in Settings > Users > Fee/fine: Manual charges — FOLIOtips 5672543).

### Triggering events and Send options [confirmed — en.json + constants.js]

| Accordion | UI label (en.json) | Stored `sendWhen` | `Send` options | Info banner |
|---|---|---|---|---|
| Loan | `Check in` | `Check in` | none | postponed |
| Loan | `Check out` | `Check out` | none | postponed |
| Loan | `Item renewed` | `Renewed` | none | real time |
| Loan | `Loan due date/time` | `Due date` | `Upon/At`, `Before`, `After` | none; required long/short-term radio |
| Loan | `Loan due date change` | `Manual due date change` | none | real time |
| Loan | `Item recalled` | `Item recalled` | none | real time |
| Loan | `Hold request for item` | `Hold request for item` | none | real time |
| Loan | `Item aged to lost` | `Aged to lost` | `Upon/At`, `After` | conditional |
| Request | `Awaiting pickup` | `Available` | none | real time |
| Request | `Page request` | `Paging request` | none | real time |
| Request | `Hold request` | `Hold request` | none | real time |
| Request | `Recall request` | `Recall request` | none | real time |
| Request | `Cancel request` | `Request cancellation` | none | real time |
| Request | `Hold shelf expiration` | `Hold expiration` | `Upon/At`, `Before` | real time |
| Request | `Request expiration` | `Request expiration` | `Upon/At`, `Before` | real time |
| Fee/fine | `Overdue fine, returned` | `Overdue fine returned` | `Upon/At`, `After` | real time |
| Fee/fine | `Overdue fine, renewed` | `Overdue fine renewed` | `Upon/At`, `After` | real time |
| Fee/fine | `Lost item fee(s) charged` | `Aged to lost - fine charged` | `Upon/At`, `After` | none; required long/short-term radio |
| Fee/fine | `Lost item returned - fee(s) adjusted` | `Aged to lost & item returned - fine adjusted` | none | real time |

Banner texts (`notice-description.js`):
- Postponed (Check in / Check out): `Always sent at the end of a session and loans are bundled into a single notice for each patron.`
- Real time: `Always sent when event is triggered.`
- Conditional (Item aged to lost): `Always sent when event is triggered and send conditions are met.`

Radio pairs (`get-real-time-labels.js`, required, none pre-selected; error `Please fill this in to continue` — C385643):
- `Loan due date/time`: `Send overnight with multiple loans/items by patron. Useful for long-term loans.` / `Send throughout the day without multiple loans/items. Useful for short-term loans.`
- `Lost item fee(s) charged`: `Send overnight with multiple lost item fee charges by patron.` / `Send throughout the day with one lost item fee charge per notice.` (en.json strings end with a period; C385643 quotes them without it — assert the en.json text).

**Validation** (`section-config-generator.js`): Template, Format, Triggering event are always required; `Send` is required when shown; durations (`by`, `and every`) must be integer 1–100 (`settings.validate.isFromOneToHundred` = `The value must be between 1 and 100`); interval and Frequency required when shown. BE rejects `frequency` on a scheduled loan notice whose Send is not Before/After, `sendHow` on non-scheduled loan notices, and `sendEvery` with `One time` (422 `This option is not valid for selected Send/Frequency` — `PatronNoticePoliciesAPI.java`).

**Delete guard:** `Cannot delete patron notice` / `This notice policy cannot be deleted, as it is in use by one or more records.` UI and BE both check whether the policy id appears in the circulation rules text (BE 400 `Cannot delete in use notice policy`). Delete confirm entry label `Delete patron notice policy`.

### Notice-card reordering — implemented (UICIRC-1355, UICIRC-1372; closed 2026-08/09)

- `Add notice above` inserts a new empty card directly above the clicked card; accordion-level `Add notice` appends at the bottom; `Notice {counter}` labels renumber top-to-bottom (C1464321).
- `Delete notice` removes that card; deleting the last card returns the accordion to its empty state with only `Add notice` (C1464323).
- Drag handle (tooltip `Drag & drop`) reorders cards within one accordion only; numbers re-sequence; order persists after `Save & close` and reopen (C1464326, C1464328, C1464330).
- Controls appear on new and existing policies and in all three accordions (C1464320, C1464322, C1464327). Adding/deleting and re-adding without saving leaves the form unchanged (C1464319).
- View-only user (`Settings (Circulation): Can view notice policies`): policy opens read-only, no Edit action, no add/delete/drag controls (C1464324, C1464331).

---

## TLR patron notice templates

`Settings > Circulation > Title level requests` → `Notice templates`: `Confirmation notice`, `Cancellation notice`, `Expiration notice` (en.json `settings.titleLevelRequests.*`). Only `Request`-category templates can be selected (FOLIOtips 5672543). Title-level hold creation/cancellation notices come from here, not from the policy (docs.folio.org settings_circulation.md "Request notices triggering events"). Details: circulation-settings.md.

---

## Runtime behaviour (backend)

- **Timers** (mod-circulation ModuleDescriptor `_timer`): loan scheduled notices every 5 min; due-date not-real-time, request, fee/fine, overdue-fine, fee/fine not-real-time scheduled notices every 2 min; patron action session expiration by timeout every 3 min. docs.folio.org says "scanned every five minutes" — allow up to ~5 min plus mail latency in expected results.
- **Sessions:** Check in / Check out notices are queued as patron action sessions (`Check-out` / `Check-in`) and sent when the session ends (`End session` or automatic session end) — `POST /circulation/end-patron-action-session`; storage `patron-action-session-storage` (C347621–C347625, C347879).
- **Grouped context:** Check in / Check out notices always use the grouped `loans` context, even for one item. Item, Effective location and Loan tokens must be inside `{{#loans}}…{{/loans}}`, otherwise only User tokens are filled (CIRC-2702 closed as not a bug; FOLIOtips 5672543). Renewal notices use a single-loan context, so the same template works without the loop (CIRC-2702).
- **Format resolution** (mod-circulation `NoticeFormat.java`, `UserPreferredNoticeFormats.java`, CIRC-2661): `Email` → channel `email` / `text/html`; `SMS` → `sms` / `text/plain`; `Print` → `mail` / `text/html`. Patron preferred contact type ids map 002 → Email, 001 → Print (mail), 003 → SMS.
- **Short title** (mod-template-engine `TemplateContextPreProcessor.java` `ITEM_SHORT_TITLE_KEY = "item.shortTitle"`, `NEWS.md`, MODTEMPENG-137): the code key is `{{item.shortTitle}}` (the Jira summary and the UI token call it `item.titleShort`; MODTEMPENG-141 Open tracks the empty real-notice rendering) = `item.title` if ≤ 25 characters, else first 25 characters + `…` (U+2026); generated only when the template references it; empty, no error, when the title is missing or blank (C1538649–C1538651).
- **Template engine:** `POST /template-request/preview` renders without saving (MODTEMPENG-135; barcode images out of scope); Handlebars resolver available alongside Mustache (`templateResolver`, MODTEMPENG-136; unsupported resolver → 400).
- **Fee amounts** are rendered as scaled strings (`toScaledString()` in `TemplateContextUtil.java`) — 2 decimal places (C357041).
- **Scheduled notice hygiene:** a scheduled notice whose loan was charged a lost item fee before the aged-to-lost notice ran is skipped as irrelevant (CIRC-2679); after an unexpected processing/cleanup error the scheduled-notice record is deleted so it is not re-sent (CIRC-2664 fix); notices for closed anonymized loans are skipped silently (CIRC-1891).

---

## API surface (endpoints + permissionsRequired)

| Module | Method + path | permissionsRequired |
|---|---|---|
| mod-template-engine (`template-engine` 2.3) | GET `/templates` · POST `/templates` · GET/PUT/DELETE `/templates/{templateId}` | `templates.collection.get` · `templates.item.post` · `templates.item.get` / `.put` / `.delete` |
| mod-template-engine | POST `/template-request` · POST `/template-request/preview` | `template-request.post` · `template-request.preview.post` |
| mod-circulation-storage (`patron-notice-policy-storage` 0.13) | GET/POST `/patron-notice-policy-storage/patron-notice-policies` · GET/PUT/DELETE `…/{id}` | `circulation-storage.patron-notice-policies.collection.get` / `.item.post` / `.item.get` / `.item.put` / `.item.delete` |
| mod-circulation-storage (`scheduled-notice-storage` 0.6) | GET/POST/DELETE `/scheduled-notice-storage/scheduled-notices` · GET/PUT/DELETE `…/{id}` | `scheduled-notice-storage.scheduled-notices.*` |
| mod-circulation-storage (`patron-action-session-storage` 0.3) | `/patron-action-session-storage/patron-action-sessions[/{id}]`, GET `/patron-action-session-storage/expired-session-patron-ids` | `patron-action-session-storage.*` |
| mod-circulation | GET `/circulation/rules/notice-policy` · `/circulation/rules/notice-policy-all` · POST `/circulation/end-patron-action-session` | `circulation.rules.notice-policy.get` · `circulation.rules.notice-policy-all.get` · `circulation.end-patron-action-session.post` |
| mod-notify (`patron-notice` 1.2) | POST `/patron-notice` | `patron-notice.post` |

Template list query used by the UI: `cql.allRecords=1 AND category=""` (all categories). Policy schema (`ramls/patron-notice-policy.json`): required `name`; each notice requires `templateId`, `format`, `realTime`, `sendOptions.sendWhen`. BE-only enum values not exposed in UI: format `Print`; fee/fine `Aged to lost & item replaced - fine adjusted`. Scheduled-notice `triggeringEvent` enum also contains `Due date - with reminder fee` and `Title level request expiration`.

---

## Capability Sets (Eureka)

| Action | Permission display name (`package.json`, en.json `permission.settings.*`) | Eureka capability set |
|---|---|---|
| View templates | `Settings (Circulation): Can view patron notice templates` | (capability set name not confirmed — verify) |
| Create/edit/delete templates | `Settings (Circulation): Can create, edit and remove patron notice templates` | `Settings - UI-Circulation Settings Notice-Templates - manage` — confirmed by TestRail C1453637 precondition (spelled `settings - UI-Circulation Settings Notice-Templates - manage` there); not present in Cypress `capabilitySets.js` (only `UI-Circulation Settings Loan-Policies` / `Other-Settings` exist); UICIRC-1232 writes `settings.UI-Circulation Settings Notice-Templates.manage` |
| View policies | `Settings (Circulation): Can view notice policies` | (capability set name not confirmed — verify) |
| Create/edit/delete policies | `Settings (Circulation): Can create, edit and remove notice policies` | (capability set name not confirmed — verify) |

Sub-permissions (`package.json`): view-notice-policies = `circulation.rules.get`, `circulation-storage.patron-notice-policies.collection.get` / `.item.get`, `templates.collection.get` / `.item.get`, `users.collection.get` / `.item.get`, `settings.circulation.enabled`; notice-policies adds `.item.post/.put/.delete`. view-notice-templates = `circulation-storage.patron-notice-policies.collection.get`, `templates.collection.get` / `.item.get`, `settings.circulation.enabled`, `users.collection.get`; notice-templates adds `templates.item.post/.put/.delete`, `template-request.post`.

---

## ECS / Multi-Tenant Notes

- ECS notice setup (C1322907–C1322909): templates, notice policy and the circulation rule with that policy live in the **Central** tenant (shared loan type / patron group created in Consortium manager); the **Member** tenant rule allows requests and uses a notice policy with no notices; patron is a Central-tenant `Patron` user; item on a shared instance in a Member tenant; check-ins switch affiliation between Member and Central.
- Central-tenant ECS request notices have limited item/title token data by design; item barcode and title must always be present (CIRC-2639 closed as design limitation; C1322907 step note). Notices are sent when the Primary request is created in Requests, not when a Mediated request is created or confirmed (CIRC-2639).
- ECS TLR: request-creation notice is sent once the placeholder patron is resolved to the real one (CIRC-2269).
- Hold shelf expiration notice was sent twice on Trillium ECS TLS — regress single delivery (CIRC-2664, C1322907).

---

## Existing automated coverage

**Cypress (stripes-testing):**
- Settings: `cypress/e2e/settings/circulation/patron-notices/` — `create-edit-delete-patron-notice-template.cy.js`, `create-edit-delete-patron-notice-policies.cy.js`, `patron-notice-template-format-and-tokens.cy.js` (C1453636, C1453637), `patron-notice-policy-add-above-delete-notice.cy.js` (C1464319–C1464323), `patron-notice-policy-reorder-notices-drag-and-drop.cy.js` (C1464326–C1464330), `patron-notice-policy-view-only-notice-controls.cy.js` (C1464324, C1464331).
- Delivery flows: `cypress/e2e/patron-notices/*.cy.js` (check-in/check-out sessions, triggers for aged to lost, recall/awaiting pickup/hold shelf expiration, page/cancel request, overdue fine returned/renewed, manual fee/fine notices, multi-options lost item fee, duplication, metadata, loan.additionalInfo, pickup notices), `cypress/e2e/recieve-notice/*.cy.js`, `cypress/e2e/circulation-log/filter-notice-send.cy.js`, `cypress/e2e/fees-&-fines/*notice*.cy.js`.
- Fragments: `cypress/support/fragments/settings/circulation/patron-notices/{newNoticePolicy,newNoticePolicyTemplate,noticePolicies,noticeTemplates}.js`.

**Karate (folio-integration-tests):** `mod-template-engine/.../features/templates.feature` (CRUD, 400 unsupported resolver, 422 validation, 404) and `templateRequest.feature` (200, 400 missing template / unsupported output format / missing language, 422); `mod-notify/.../features/patronNotice.feature` (multiple-loans notice). No Karate feature covers `patron-notice-policy-storage` CRUD/validation, the in-use delete guard, `item.shortTitle`, or scheduled-notice timers. MODTEMPENG-137 was verified with Postman cases C1538649–C1538651.

---

## Patron Notice Template Duplication [confirmed — C396392]

Actions > Duplicate on an existing template redirects to the "New patron notice template" page, pre-filled with the source's Description and Category and with "Active" checked by default. The Name field is pre-filled with the SOURCE name and immediately shows `A patron notice with this name already exists`, blocking Save & close until renamed. Before saving, the duplicate's own metadata accordion shows Source = "Unknown user" (expected pre-save state). Policies can be duplicated the same way; two policies cannot share a name (docs.folio.org settings_circulation.md).

## `loan.additionalInfo` Token — Always Resolves to the Latest Comment [confirmed — C414969]

When a loan has multiple patron/staff comments (see loans.md "Additional Loan Comments"), `{{loan.additionalInfo}}` in any notice — immediate (e.g. Loan due date change) or scheduled (e.g. Loan due date/time) — resolves only to the most recent, non-superseded comment. Parsing relies on the words STAFF and PATRON; tenants that translate them get no comment in notices (FOLIOtips 5672543, UXPROD-4106).

## Proxy / Sponsor Notification Routing [confirmed — C784435–C784438, CIRC-2385]

A proxy relationship carries "Notifications sent to" = **Proxy** or **Sponsor**. It governs every Loan notice (Check out, Check in, …) and Request notice (Page request, Hold request, Awaiting pickup, Cancel request, …) triggered by activity "acting as proxy for" that sponsor: Proxy → only the proxy's mailbox; Sponsor → only the sponsor's mailbox. No per-template or per-policy override.

## Anonymized / Closed Loans [confirmed — C729196, CIRC-1891]

If a scheduled notice is still pending when the loan is closed and anonymized, no notice is sent and no "Send error" entry appears in the Circulation log.

## Overdue-Fine Notice Bundling — Precise Scope [confirmed — C397371, C397372, C397373, C397356, C397380]

Multiple overdue-fine charges resolved in one Check-in or Users-app session bundle into a single email via `{{#feeCharges}}…{{/feeCharges}}` when they share the same triggering event ("Overdue fine, returned" or "Overdue fine, renewed"), for both `After` and `Upon/At`. Bundling never crosses event types: one item renewed + another returned → two emails.

## Fee Amount Token — Always 2 Decimal Places [confirmed — C357041]

`{{feeCharge.amount}}` / `{{feeCharge.remainingAmount}}` always render with two decimals (e.g. `110.90`), same as the Loan-details "Fees/fines incurred" field.

## `item.displaySummary` Token [confirmed — C449392]

Listed immediately before `item.enumeration`. Preview sample `no.1-2v. 121948-1962 (Board)` (also `tokens.js`).

## Awaiting-Pickup Notices Are Scoped Per-Request [confirmed — C400642]

Each requester whose item becomes Awaiting pickup gets their own notice; notices are never merged across requesters, even under identical policy/rule.

---

## Key Business Rules for Test Cases

1. A notice is produced only when all four exist: an active template, a notice policy referencing it for the event, a circulation rule assigning that policy, and the triggering event occurring. (docs.folio.org settings_circulation.md; C1322907 preconditions)
2. Template Category constrains token availability and where the template can be selected: Loan → Loan notices; Request → Request notices and TLR notices; Automated fee/fine charge/adjustment → Fee/fine notices; Manual fee/fine categories → Settings > Users manual charges only. (NoticePolicyForm.js + get-templates.js; FOLIOtips 5672543)
3. The policy Template dropdown lists only active templates of the allowed categories; an inactive template disappears from the dropdown. (get-templates.js)
4. Notice policies split by recipient: Loan notices → borrower, Request notices → requester, plus Fee/fine notices. (en.json `settings.noticePolicy.loanNotices`, `requestNotices`)
5. Send options exist only for time-based events: Loan due date/time (Upon/At, Before, After), Item aged to lost (Upon/At, After), Hold shelf expiration and Request expiration (Upon/At, Before), Overdue fine returned/renewed and Lost item fee(s) charged (Upon/At, After). All other events have no Send/Frequency fields. (LoanNoticesSection.js, RequestNoticesSection.js, FeeFineNoticesSection.js, constants.js)
6. Frequency appears only with Before/After; Recurring adds `and every`; durations must be integers 1–100. (NoticeSendOptions.js, section-config-generator.js)
7. Loan due date/time and Lost item fee(s) charged require choosing one of two long-/short-term radios; none is pre-selected and Save is blocked with `Please fill this in to continue`. (section-config-generator.js `realTime`; C385643)
8. Check in / Check out notices are sent only at session end and always bundle all loans of the patron's session into one notice. (en.json `settings.noticePolicy.notices.postponed.notification`; C347621–C347625)
9. Check in / Check out templates must wrap Item, Effective location and Loan tokens in `{{#loans}}…{{/loans}}`; without the loop only User tokens are filled. Renewal notices work without it. (CIRC-2702; FOLIOtips 5672543)
10. Multiple fee/fine charges loop is selectable only for category Automated fee/fine charge; Multiple loans must not be used in fee/fine templates. (TokensList.js; FOLIOtips 5672543)
11. Templates referenced by any notice policy cannot be deleted (UI modal + BE InUseTemplateException); predefined templates cannot be deleted at all. (PatronNotices.js, TemplateServiceImpl.java, en.json `settings.patronNotices.predefinedWarning`)
12. A notice policy whose id appears in the circulation rules cannot be deleted (UI modal; BE 400 `Cannot delete in use notice policy`). (NoticePolicySettings.js, PatronNoticePoliciesAPI.java)
13. Template names and policy names must be unique; duplicate template name shows `A patron notice with this name already exists`; duplicate policy name gets BE 422. (en.json `settings.patronNotices.errors.nameExists`; PatronNoticePoliciesAPI.java)
14. Template Notice format is required; Body stays disabled until a format is chosen; Subject is enabled and required only for Email. (PatronNoticeEmailSection.js; patron-notice-template.js; UICIRC-1350)
15. Subject must not begin with a space. (en.json `settings.validate.subject`; C356783)
16. Switching Notice format restores the Subject/Body previously typed for that format; the first switch back to Email restores the saved template values. (resolveNoticeFormatFieldValues.js)
17. Text message templates are plain text, image tokens are disabled, and no 160-character validation is enforced in UI (only an InfoPopover). (PatronNoticeEmailSection.js; UXPROD-1532 comment 2026-05-04)
18. Saving a Text message template stores `outputFormats` = `text/plain` and an auto header `textMessage`; Print stores header `print` and `printOnly` = true. (PatronNotices.js handleBeforeSave)
19. Policy Format offers exactly `Email` and `Text message`; no default is selected for Template, Format or Triggering event. (constants.js `noticesFormats`; UICIRC-1349)
20. `Add notice above` inserts a card above the clicked one; `Add notice` appends at the bottom; `Notice {counter}` always renumbers sequentially. (UICIRC-1355; C1464321)
21. Drag & drop reorders cards only within the same accordion, and the order persists after Save & close. (UICIRC-1372; C1464328, C1464330)
22. Users with only view-notice-policies see no Edit action and no add/delete/drag controls on notice cards. (C1464324, C1464331)
23. `item.titleShort` (UI token) is listed between `item.title` and `item.primaryContributor`; preview shows the first 25 characters; a real notice using it renders empty until MODTEMPENG-141 is fixed. (UICIRC-1353; C1453636, C1453637; MODTEMPENG-141)
24. `{{item.shortTitle}}` (the key the template engine actually generates) renders the title unchanged up to 25 characters, else 25 characters + `…`; missing/blank title renders empty without error. (MODTEMPENG-137; TemplateContextPreProcessor.java; C1538649–C1538651)
25. Notice delivery format follows the policy row: Email → email (`text/html`), SMS → text message (`text/plain`), Print → mail. (NoticeFormat.java; CIRC-2661)
26. Verify a notice's real effect at the mailbox — exact Subject + Body with tokens resolved — not only the Circulation-log entry. (write-testrail-cases skill; C1322907)
27. TLR Confirmation/Cancellation/Expiration notices are selected under Title level requests settings, separate from policy request notices; title-level hold creation/cancellation use those templates. (en.json `settings.titleLevelRequests.*`; docs.folio.org settings_circulation.md)
28. Patron notice templates can be duplicated via Actions > Duplicate; the copy keeps Description/Category but needs a new unique name before saving. (C396392)
29. `loan.additionalInfo` resolves to the single most recent non-superseded loan comment. (C414969)
30. Proxy/Sponsor routing ("Notifications sent to") applies uniformly to every Loan and Request notice for that relationship; no per-template or per-policy override. (C784435–C784438; CIRC-2385)
31. Overdue-fine bundling merges only charges with the same triggering event; returned + renewed for the same patron in one window gives two emails. (C397372, C397373, C397380)
32. Fee-amount tokens always render with two decimal places. (C357041; TemplateContextUtil.java)
33. Pending scheduled notices for closed, anonymized loans are dropped silently — no Circulation-log "Send error". (C729196; CIRC-1891)
34. An aged-to-lost notice is skipped if a lost item fee was already charged before it ran; for testing keep "Patron billed after aged to lost" longer than the notice delay (C347630 uses a notice After 1 hour with the value greater than 3 hours and aged-to-lost after overdue = 1 min). (CIRC-2679; C347630)
35. A scheduled notice must be delivered once only; hold shelf expiration notices must not duplicate. (CIRC-2664; CIRC-2329)
36. "Item recalled" notices go to every borrower whose item is recalled, not only the first item in the title queue. (CIRC-2168)
37. Each requester whose item becomes Awaiting pickup gets their own independent notice. (C400642)
38. In ECS, notices are configured in the Central tenant; Central-tenant request notices may lack some item/title tokens but always carry item barcode and title; mediated requests notify only when the primary request is created. (CIRC-2639; C1322907)
39. Template record metadata (`Record created` / `Record last updated` + `Source`) updates only the last-updated pair on edit. (C411862)

---

## Authoring style (measured 2026-07-23)

Patron Notices: mixed **`Other` ~53% / `Func` ~40%**, median **~12 steps**, `User Journey` ~2%. Template/policy configuration cases lean `Other`; end-to-end "configure notice → trigger event → notice lands in mailbox with resolved tokens" cases lean `Func` and run long. Preconditions carry template + policy + circulation-rule + triggering setup (see C1322907 for the ECS pattern); the flow walks event → delivery. Scheduled triggers need waits sized to the timers (2–5 min) and to policy intervals.

---

## Known Gaps / Items to Verify

- [ ] **Token name mismatch — defect tracked in MODTEMPENG-141 (Open, 2026-11-25): "item.titleShort token is not populated when processing template for sent notices" (renders empty outside preview, both text/plain and text/html).** Resolved as to source: UI inserts `{{item.titleShort}}` (`tokens.js`, UICIRC-1353, C1453636/C1453637 check only the Preview), but mod-template-engine generates `item.shortTitle` (`TemplateContextPreProcessor.java`, MODTEMPENG-137, Postman C1538649–C1538651); mod-circulation supplies neither key. Jira MODTEMPENG-141 states the intended token is `item.titleShort` (same 25 chars + `…` rule) and the real-notice context is not populated; no code fix is in the cloned master. Do not assert a resolved value in a real notice until MODTEMPENG-141 is fixed; preview-only checks (C1453636/C1453637) remain valid.
- [ ] **"Hold request for item" loan trigger.** Present in ui-circulation master and the storage schema enum, but UICIRC-1245 is Blocked, the trigger was removed from Trillium 12.0.1 (UICIRC-1362), and mod-circulation `NoticeEventType.java` has no such event. Selecting it likely sends nothing; do not write delivery cases until BE support lands.
- [ ] **Formats settings pane / "User preference" format — not in master.** C1504502–C1504508 (refs CIRC-2661, UICIRC-149) describe a `Formats` pane and a `User preference` / `Print` policy format, but UICIRC-149 is Closed as Won't Do, en.json has no Formats keys, and the policy Format dropdown offers only Email / Text message. CIRC-2661 (BE format resolution) is Closed without a resolution. Treat those cases as unexecutable on current master until confirmed.
- [ ] Policy-level success toasts (create/update/delete notice policy) and template save toast — no en.json key is specific to notice policies/templates; wording not confirmed in any source.
- [ ] Eureka capability-set names for view templates, view policies and manage policies are not in any source read; only `Settings - UI-Circulation Settings Notice-Templates - manage` is confirmed (C1453637 precondition; not in Cypress `capabilitySets.js`).
- [ ] docs.folio.org is outdated: it still says Format "email is the only option" and describes an "Email or print" section with a "Print only" checkbox (settings_circulation.md); master UI has Notice format with Email / Print only / Text message.
- [ ] en.json keys `settings.patronNotices.multipleRequests` / `multipleFeeFineActions` exist but are not used by `TokensList.js` — do not assert those loops.
- [ ] Text message delivery end-to-end (SMS provider, URL shortening promised in UICIRC-1350) is not covered by any test case or Karate feature read; depends on MODSENDER work under UXPROD-1532 (In Review).
- [ ] TestRail C347630 (older case) describes an `Email` accordion and `Loan` as the default Category; master UI has `Patron notice content` / Notice format and Category has no verified default — treat those steps as outdated and re-verify on execution.
- [ ] TestRail C385643 quotes the Lost-item radio labels without the final period; en.json has a period — confirm on execution.
- [ ] Exact `{{…}}` token rendering for `User`, `Request`, `Fee/fine action` groups at the mailbox is confirmed only for item tokens (C692204), fee amounts (C357041) and `loan.additionalInfo` (C414969).
- [ ] Whether Proxy/Sponsor routing has any override path in newer releases was not tested (single relationship-level switch in C784435–C784438).

> Refresh 2026-11-24: folded the 2026-11-07 delta into the main sections. Corrected: policy Format options (Email / Text message, not Email only), template Notice format options (Email / Print only / Text message — the delta listed only Email / Text message), template accordion name (`Patron notice content`), source of UI strings (`en.json`, not `en_US.json`), reorder controls (implemented, view-only behaviour confirmed by C1464324/C1464331). Added: Send-options matrix, token availability by category, validation rules, API surface, timers, ECS notes, automated coverage, BE-only enum values, and 25 new rules (15–39 plus rewritten 1–14).
