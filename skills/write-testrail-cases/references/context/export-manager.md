---
area: export-manager
generated_on: 2026-11-24
generator_version: manual (build-app-context method, full refresh)
jira_status: ok
---

# FOLIO Export Manager (+ EDIFACT / CSV order and claim export) — Business Logic Context

> Built: 2026-11-07 · Refreshed: 2026-11-24 · Release: Umbrellaleaf R2 2026 (master: ui-export-manager 5.1.0 in progress, mod-data-export-spring 3.7.0 unreleased, mod-data-export-worker 3.6.0 unreleased; Trillium R1 2026 = ui-export-manager 5.0.0, UIEXPMGR-138) · Builder: build-app-context method, manual
> Sources:
> - TestRail suite 21: section 11061 "Export Manager" (18 cases), 17492 "Export Orders in EDIFACT format" (2), 17493 "Orders Export to a Vendor" (45), 72797 "Settings (Export manager)" (1); details read for C405555, C813000, C784505, C1263891.
> - GitHub (HEAD 2026-11): `ui-export-manager` (src/, en.json 60 keys = en_US.json, package.json, CHANGELOG); `mod-data-export-spring` (descriptors/ModuleDescriptor-template.json, swagger.api/, JobServiceImpl, validators, Liquibase changelog, NEWS.md); `mod-data-export-worker` (acquisitions tasklets, NEWS.md); `folio-export-common` (schemas); `stripes-acq-components` (lib/constants/export.js); `folio-integration-tests` (acquisitions/thunderjet/mod-data-export-spring); `stripes-testing` (cypress/e2e/export-manager, fragments).
> - Jira (anonymous REST): UIEXPMGR (35 issues since 2024-06), MODEXPS Done/Closed stories+bugs since 2024-09 (23), MODEXPW Done/Closed export-related since 2025-01 (37); details read for ~30.
> - docs.folio.org: Settings > Export manager (last modified 2026-06-30).
> - Related context files: `orders.md` (automatic export, acquisition method), `organizations.md` (integration details, scheduling), `claiming.md`.
> Tags: [source] = GitHub only; [cases] = TestRail only; [confirmed] = both.

## What is Export Manager

The **Export manager** app (`meta.title` "Export manager") lists **export jobs** produced by other apps and schedulers: Circulation log, Bursar (Fees/fines), eHoldings, Authority control (MARC authority headings updates, Failed updates: Linked bibliographic fields), and acquisitions integrations (EDIFACT / CSV **orders** and **claims**). Two tabs: **All** (route `/export-manager/jobs`) and **Organizations** (route `/export-manager/edi-jobs`, vendor integration jobs). Jobs can be opened in a third pane and files downloaded; organization jobs can also be **rerun** and **resent**. Settings > Export manager > Jobs holds per-job-type **deletion intervals**. Bulk edit jobs are no longer shown (UIEXPMGR-128, MODEXPS-297).

Backend: `mod-data-export-spring` (jobs, configs, scheduling, deletion) + `mod-data-export-worker` (Spring Batch execution, file generation, FTP/email delivery, S3/MinIO storage).

## Key Terms

| Term | Definition |
|---|---|
| Export job | Record in `data-export-spring/jobs`; `name` is the Job ID shown in UI (6-digit zero-padded sequence, `%06d`) (JobServiceImpl) |
| Export type (BE) | `CIRCULATION_LOG`, `BURSAR_FEES_FINES`, `EDIFACT_ORDERS_EXPORT`, `CLAIMS`, `E_HOLDINGS`, `AUTH_HEADINGS_UPDATES`, `FAILED_LINKED_BIB_UPDATES` (+ unused here: `BATCH_VOUCHER_EXPORT`, `ORDERS_EXPORT`, `INVOICE_EXPORT`) (folio-export-common schemas/exportType.json) |
| Integration (export config) | Vendor integration from Organizations > Integration details, stored in `data-export-spring/configs`; `integrationType` Ordering / Claiming; `fileFormat` EDI / CSV; `transmissionMethod` FTP / File download / Email (vendorEdiOrdersExportConfig.json) |
| Default integration | `isDefaultConfig=true` Ordering integration: exports POLs of the vendor whose account number is not listed on another (non-default) integration; needs no account numbers (MODEXPS-322, MODEXPW-663/664) |
| System source | `isSystemSource=true` → Source column shows "System" (scheduled runs); manual execution and user-started jobs are `false` (MODEXPS-316) |
| Deletion interval | `retentionDays` per export type in `data-export-spring/job-deletion-intervals` |

## Job Model and Lifecycle

### Status values (en.json `exportJob.status.*`, schemas/jobStatus.json)
Scheduled → In progress → Successful | Failed. New jobs default to `SCHEDULED` (JobServiceImpl.upsert).

### Job types
- **All tab "Job type" filter** (src/ExportJobs/constants.js): Authority control, Bursar, Circulation log, eHoldings, Orders (EDI), Orders (CSV).
  - "Authority control" requests both `AUTH_HEADINGS_UPDATES` and `FAILED_LINKED_BIB_UPDATES` (EXPORT_JOB_TYPES_REQUEST_MAP).
  - "Orders (EDI)" / "Orders (CSV)" = type `CLAIMS` or `EDIFACT_ORDERS_EXPORT` with `fileFormat` EDI / CSV — i.e. these filters return **claims jobs too** (src/ExportJobs/apiQuery.js + stripes-acq-components ORGANIZATION_INTEGRATION_EXPORT_TYPES; C784505 step 4).
- **Job type column / detail** uses `exportJob.details.type.*`: "Circulation log", "Bursar", "EDIFACT orders export", "eHoldings", "MARC authority headings updates", "Failed updates: Linked bibliographic fields"; unknown keys fall back to the raw type (`defaultMessage`) — no key exists for `CLAIMS` (ExportJobsList.js) [source].
- Output type (`outputFormat`) is set by BE: Bursar "Fees & Fines Bursar Report", Circulation log "Comma-Separated Values (CSV)", EDIFACT orders "EDIFACT orders export (EDI)"; max 50 chars (MODEXPS-287/291) (JobServiceImpl).

### Generated file names (worker ExportUtils.generateFileName)
`<prefix>_<vendor code>_<integration name>_<yyyy-MM-dd_HH-mm-ss>.<edi|csv>`; prefix `edi_orders`, `csv_orders`, `edi_claims`, `csv_claims`. No colon in the timestamp (MODEXPW-435).

## UI Structure

### All tab
- Filters (accordions): Status, Job type, System (boolean), Source (user plugin), Start time, End time (date ranges) (ExportJobsFilters.js).
- Columns: Job ID, Status, Job type, Description, Source, Start time, End time; default sort Job ID descending; Prev/Next pagination (ExportJobsList.js; C365128).
- Search: query matches `name` (Job ID) or `description` (apiQuery.js).
- Third pane "Export job {jobId}": Job ID, Status, Job type, Output type, Description, Source, Start time, End time, Error details + record metadata (ExportJob.js). No Actions menu.

### Organizations tab
- Shown only when the `organizations.organizations` interface is present; otherwise `/export-manager/edi-jobs` redirects to `/export-manager/jobs` (UIEXPMGR-127, ExportManager.js).
- Filters: Status, Integration type (Claims / Orders), Export method (integration names; options narrowed to the vendor selected in Organization), Organization, System, Source, Start time, End time (ExportEdiJobsFilters.js; UIEXPMGR-115; C350596, C359175).
- Columns: Job ID, Status, Description, Source, Start time, End time, Export method (ExportEdiJobsList.js).
- Third pane: Job ID, Status, Start time, End time, Source, Organization, Export method, Sent to, File name, Description, Error details (ExportEdiJobDetails.js; UIEXPMGR-116; C405555).
  - "Sent to" = "Download" when transmission method is File download, else `<serverAddress><orderDirectory>` [source].
  - Actions menu (only with `ui-export-manager.export-manager.all`): **Rerun** (needs `data-export.job.item.post`; disabled if the integration no longer exists), **Download** and **Resend** (only if the job has file names; need `.download` / `.resend`) (ExportEdiJobDetailsActionMenu.js; C359196).

### Settings > Export manager > Jobs
- Pane "Jobs", legend "Job deletion intervals (days)", one numeric field per export type, sorted alphabetically by label: Authority control, Bursar, Circulation log, Claims, EDIFACT orders export, eHoldings (JobsSettings.js, constants.js; stripes-testing settings constants; docs.folio.org).
- Save button (stripes-acq-components `button.save` "Save") disabled while pristine, submitting or for view-only users; fields non-interactive without `ui-export-manager.settings.edit` (JobsSettingsForm.js; C813000).

## API Surface (mod-data-export-spring, interface `data-export-spring` 2.1)

| Method & path | permissionsRequired | Notes |
|---|---|---|
| GET `/data-export-spring/jobs` | data-export.job.collection.get | CQL `query`, `offset`, `limit` |
| GET `/data-export-spring/jobs/{id}` | data-export.job.item.get | 404 "Job {id} not found" |
| POST `/data-export-spring/jobs` | data-export.job.item.post | create / rerun (UI posts `type` + `exportTypeSpecificParameters`) |
| GET `/data-export-spring/jobs/{id}/download` | data-export.job.item.download | `?key=` for multi-file (Bursar); 404 if no file names |
| POST `/data-export-spring/jobs/{id}/resend` | data-export.job.item.resend | 404 "The exported file is missing for jobId: …" if no file |
| POST `/data-export-spring/jobs/send` | data-export.job.send.item.execute | |
| GET/POST `/data-export-spring/configs`, GET/PUT/DELETE `/configs/{id}` | data-export.config.collection.get / .item.get / .item.post / .item.put / .item.delete | integrations; validation errors 400 |
| POST `/data-export-spring/configs/{id}/execute` | data-export.config.item.execute | body `{poLineIds}` → 201 `{jobId}`; 404 unknown config; 400 `configNotExecutable` if not Ordering EDIFACT_ORDERS_EXPORT; 400 empty poLineIds (MODEXPS-316) |
| GET `/data-export-spring/effective-configs` | data-export.effective-configs.collection.get | `vendorId`, `type` (EDIFACT_ORDERS_EXPORT / CLAIMS) required; optional acquisitionMethod, vendorAccount, transmissionMethod, scheduled (MODEXPS-323) |
| GET `/data-export-spring/job-deletion-intervals`, GET `/{exportType}` | data-export.job-deletion-intervals.collection.get / .item.get | 404 if type missing |
| POST `/data-export-spring/job-deletion-intervals` | data-export.job-deletion-intervals.item.post | 409 if type exists |
| PUT / DELETE `/data-export-spring/job-deletion-intervals/{exportType}` | data-export.job-deletion-intervals.item.put / .item.delete | `retentionDays` required, minimum 1 |

Source: descriptors/ModuleDescriptor-template.json, JobServiceImpl, JobDeletionIntervalServiceImpl, schemas/jobDeletionInterval.json.

## Permissions and Capability Sets

| Okapi permission (package.json) | displayName (en.json) | Sub-permissions (key) | Eureka capability set (from cases) |
|---|---|---|---|
| ui-export-manager.export-manager.view | Export manager: View | data-export.job.collection.get/.item.get, data-export.config.*.get, organizations, users, tags | data - UI-Export-Manager Export-Manager - view (C1263891) |
| ui-export-manager.jobs.downloadAndResend.execute | Export manager: Download and resend files | data-export.job.item.post, .item.download, .item.resend | procedural - UI-Export-Manager Jobs DownloadAndResend - execute (C1263891) |
| ui-export-manager.export-manager.all | Export manager: All | view + downloadAndResend + data-export.edifact.orders.create | data - UI-Export-Manager Export-Manager - manage (C784505) |
| ui-export-manager.settings.view | Settings (Export manager): Can view only settings | job-deletion-intervals.collection.get | settings - UI-Export-Manager Settings - view (C813000) |
| ui-export-manager.settings.edit | Settings (Export manager): Can view and edit settings | settings.view + job-deletion-intervals.item.put | settings - UI-Export-Manager Settings - edit (C784505) |

## ECS / Multi-Tenant Notes

- Deletion intervals and jobs are per tenant; deletion runs per tenant (C813001).
- With Central Ordering enabled, an EDIFACT export of a central-tenant order whose POL uses a member-tenant location succeeds and includes that location (MODEXPW-603, C825295).

## EDIFACT / CSV order export rules

1. A scheduled orders export selects POLs where: PO workflow status is Open, PO vendor = integration vendor, PO is not Manual, POL `automaticExport` = true, POL not exported before (`lastEDIExportDate` empty), POL acquisition method is in the integration's acquisition methods, and the vendor-account filter matches (worker MapToEdifactOrdersTasklet.getPoLineQuery; C350395–C350404, C350604).
2. An already exported POL is not exported again by later runs (lastEDIExportDate) (Karate `exported-order-not-repeated-in-next-exports.feature`).
3. Vendor-account filter: non-default integration → POL account in its account list; default integration → POL account not listed on any non-default Ordering integration of the vendor; account numbers left on default integrations are ignored; if no non-default integration has accounts, no account filter (MODEXPW-663, MODEXPW-664; C422006).
4. Manual execution (`POST /configs/{id}/execute`): restricted to the given POL ids, ignores the "PO not Manual" and "Automatic export" conditions, all other filters still apply; non-matching lines are silently dropped; job is `isSystemSource=false` (MODEXPS-316, MODEXPW-652).
5. If no POLs match, the job fails with error details "Entities not found: PurchaseOrder" (worker NotFoundException; MODEXPW-664) [source].
6. Opening an order whose POLs map to more than one vendor account for export shows an alert (C350410, C350433, C350434); no alert when automatic export is disabled on the POLs (C350437).
7. Two integrations of one organization cannot share an account number (C350552).
8. EDI integration (non-default) requires vendor account number(s); EDI requires Library and Vendor EDI codes; FTP requires server address and port; CSV ordering integrations do not need account numbers (AbstractExportParametersValidator; MODEXPW-628; MODEXPS-322).
9. Scheduling: periods Hourly / Daily / Weekly / Monthly (BE enum MONTH, WEEK, DAY, HOUR, EXACT_DATE, NONE); schedule time must be `HH:mm:ss`; export runs at the time chosen in the UI; disabled scheduler stops jobs (scheduleParameters.json, EdifactOrdersScheduledParamsValidator; C350595, C359156, C359201, C360088, C360525, C360527, C423526, C423529, C359161).
10. Transmission methods: FTP, File download, Email (Email: from, to, template required; BCC supported) (MODEXPW-625, MODEXPW-642, ExportUtils.validateEmailFields).
11. Failed FTP upload or email delivery fails the job and does not write export history, so the POLs stay eligible for the next run (MODEXPW-660).
12. EDIFACT message: UNH and UNT reference numbers match (C430228); a currency different from the default is exported (C523775); PRI+AAB contains only the price, currency is in CUX (MODEXPW-596).
13. EDI claim files use the OSTENQ message, 91 qualifiers, PIA/STS/RFF+SNA/RFF+ACT segments (MODEXPW-584).

---

## Key Business Rules for Test Cases

1. Both tabs show no jobs until a filter is selected or a search is entered; the empty pane shows "Choose a filter or enter a search query to show results" (apiQuery.js `enabled: !filtersEmpty`; C380489, C405555).
2. The "All" tab is selected by default; "Organizations" tab is visible but not selected (C405555).
3. The Organizations tab is hidden and its route redirects to the All tab when the `organizations.organizations` interface is absent (UIEXPMGR-127; ExportManager.js).
4. Job ID is a download link only when the job has files AND the user has `Export manager: All`; "View" alone, and "View" + "Download and resend files", show a disabled (plain text) Job ID in the list and in the third pane (ExportJobId.js; C405555, C1263891; UIEXPMGR-132/135 closed Won't Do).
5. The Organizations third-pane Actions menu is hidden without `Export manager: All` (ExportEdiJobDetails.js; C405555).
6. Rerun / Download / Resend each also require the matching BE permission (`data-export.job.item.post` / `.download` / `.resend`); Download and Resend appear only when the job has file names (ExportEdiJobDetailsActionMenu.js).
7. Rerun creates a new job with the same type and parameters, shows "Export job <b>{name}</b> was successfully rerun." and opens the new job; failure shows "Export job <b>{name}</b> rerun was failed." (en.json `exportJob.details.action.rerun.*`; C784505).
8. Rerun is disabled when the job's integration was deleted (isRerunDisabled on config load error; C359196).
9. Resend re-uploads the exact stored file of the job (also for Failed jobs) and shows "Your file upload has started successfully."; failure shows "Your file upload was failed." (en.json; C365604; UIEXPMGR-112).
10. Resend works only for FTP integrations; for other transmission methods the resend fails ("Transmission method must be FTP in order to resend exported file") (worker ResendService; UIEXPMGR-116 "Resend error").
11. Download returns the exact generated file, also for Failed jobs and files older than one day; download failure shows "Your file download was failed." (C365605, C368040, C368493, C369079; useSecureDownload.js).
12. Bursar jobs can have several files; each is downloaded separately via `download?key=` (MULTIPLE_EXPORTED_JOB_TYPES; UIEXPMGR-66).
13. "Authority control" filter returns both "MARC authority headings updates" and "Failed updates: Linked bibliographic fields" jobs (constants.js; C1263891).
14. "Orders (EDI)" / "Orders (CSV)" on the All tab include claims jobs of that file format (apiQuery.js; C784505).
15. Organizations "Integration type" filter has options Orders and Claims (en.json `exportJob.integrationType.*`; UIEXPMGR-115).
16. "Export method" filter lists all existing integrations (C359175; MODEXPS-282) and sorting by Export method does not reset results (C377045; UIEXPMGR-81).
17. Source shows "System" for scheduled jobs (`isSystemSource`), otherwise the user name (ExportJobsList.js; MODEXPS-316).
18. Default sort is Job ID descending; Description and Source are sortable (ExportJobsList.js; UIEXPMGR-110; C552369).
19. Results use Next / Previous pagination on both tabs (C365128, C365586; UIEXPMGR-54).
20. Job deletion runs daily per tenant and deletes jobs whose last update is older than the interval for their type, comparing at midnight, so jobs disappear one day after the interval (JobServiceImpl.deleteOldJobs, OldJobDeleteScheduler; UIEXPMGR-129 comment; C784505).
21. Default deletion intervals: Authority control, Bursar, Circulation log, eHoldings = 7 days; EDIFACT orders export, Claims = 730 days (Liquibase 17_07_2025_create_job_deletion_interval.xml; docs.folio.org; C813000 values).
22. Deletion interval is required, must be a positive whole number (minimum 1); there is no upper limit (validateRequiredPositiveNumber; jobDeletionInterval.json; UIEXPMGR-129 comment).
23. Saving shows one success toast for all changed rows: "Deletion interval for <b>{entities}</b> was successfully updated." / plural "Deletion intervals for … were successfully updated."; failed rows get a separate error toast; only changed rows are sent (PUT per type) (JobsSettings.js; en.json `settings.jobs.update.*`).
24. View-only settings users see read-only fields and an inactive Save button (C813000).
25. Default Ordering integration can be saved without vendor account numbers (MODEXPS-322; C402381).
26. Claims jobs appear on the Organizations tab with Integration type = Claims and support Rerun / Download / Resend like orders (UIEXPMGR-116; see `claiming.md`).
27. A download-only (File download) claims job ends Successful, not Failed (MODEXPW-555).
28. An export job whose delivery failed is Failed, and its POLs are exported again by the next run (MODEXPW-660).
29. Authority control CSV reports export with any tenant language (outputFormat ≤ 50 chars) (MODEXPS-291).
30. Bulk edit jobs are not listed; old BULK_EDIT_* records in the DB do not break Search & filter (MODEXPS-297, MODEXPS-312; C350727).

---

## Exact UI Texts

> From `folio-org/ui-export-manager` `translations/ui-export-manager/en.json` at HEAD (commit 6efc297, 2026-09-15; en_US.json identical, 60 keys). `{x}` are placeholders; `<b>` renders bold. Use verbatim.

### Toast Messages

| Key | Text | Source |
|---|---|---|
| `exportJob.details.action.download.error` | Your file download was failed. | [source] |
| `exportJob.details.action.rerun.success` | Export job <b>{name}</b> was successfully rerun. | [source] |
| `exportJob.details.action.rerun.error` | Export job <b>{name}</b> rerun was failed. | [source] |
| `exportJob.details.action.resend.success` | Your file upload has started successfully. | [source] |
| `exportJob.details.action.resend.error` | Your file upload was failed. | [source] |
| `settings.jobs.update.success` | {count, plural, one {Deletion interval for <b>{entities}</b> was successfully updated} other {Deletion intervals for <b>{entities}</b> were successfully updated}}. | [source] |
| `settings.jobs.update.error` | {count, plural, one {Deletion interval for <b>{entities}</b> was not updated} other {Deletion intervals for <b>{entities}</b> were not updated}}. | [source] |

### Pane titles, tabs, actions

| Key | Text |
|---|---|
| `meta.title` | Export manager |
| `title.all` / `navigation.all` | All |
| `title.organizations` / `navigation.organizations` | Organizations |
| `exportJobs` | Export jobs |
| `exportJobs.search` | Search |
| `exportJob` | Export job {jobId} |
| `exportJob.details.action.rerun` | Rerun |
| `exportJob.details.action.download` / `exportJob.download` | Download |
| `exportJob.details.action.resend` | Resend |
| `settings.title` | Export manager settings |
| `settings.jobs` | Jobs |
| `settings.jobs.deletionIntervals` | Job deletion intervals (days) |

### Field / column / filter labels
Job ID (`exportJob.jobId`), Status, Job type, Output type, Description, Source, Start time, End time, Error details, System (`exportJob.system`), Organization, Export method, Sent to, File name, Integration type; integration types Claims (`exportJob.integrationType.claiming`), Orders (`exportJob.integrationType.ordering`).

### Status / type values
- Status: Scheduled, In progress, Successful, Failed (`exportJob.status.*`).
- Job type filter: Circulation log, Bursar, eHoldings, Authority control, Orders (EDI), Orders (CSV) (`exportJob.type.*`).
- Job type detail: Circulation log, Bursar, EDIFACT orders export, eHoldings, MARC authority headings updates, Failed updates: Linked bibliographic fields (`exportJob.details.type.*`).

### Permission display names
Export manager: View; Export manager: Download and resend files; Export manager: All; Settings (Export manager): Can view only settings; Settings (Export manager): Can view and edit settings (`permission.*`).

### From cases / shared components
- Empty results: "Choose a filter or enter a search query to show results" (C405555; stripes-acq-components NoResultsMessage) [cases].
- Settings Save button: "Save" (stripes-acq-components `button.save`).

---

## Existing Automated Coverage

### Karate (folio-integration-tests, `acquisitions/src/main/resources/thunderjet/mod-data-export-spring/`)
Runner `data-export-spring.feature` (Java: `DataExportSpringApiTest`, `DataExportSpringExtendedApiTest`); features: `edifact-orders-export`, `claims-export`, `claims-export-acquisition-method`, `exported-order-not-repeated-in-next-exports` (also EDI special-character escaping), `automatic-export-flag-triggers-edifact-export`, `edifact-export-manual-disabled-pol-automatic-export-enabled`; reusables `create-integration-details`, `resend-export-job-file`, `verify-export-job-file`. Not covered by Karate: job-deletion-intervals API, `/configs/{id}/execute`, `/effective-configs`, email transmission.

### Cypress (stripes-testing)
- Specs: `cypress/e2e/export-manager/` (landing page, search-filter, multiple filters, view, check-view-jobs, cannot-view, cannot-download-eholdings-and-authority-control-files (C1263891), schedule-time, job-for-order-with-acquisition-unit, manage-transfer-permissions), `export-orders-in-edifact-format/` (job type filter options, resend failed job, sort by export method), `orders-export-to-a-vendor/` (default SFTP integration, download exact .edi, scheduling daily/weekly, multi-account alerts, currency, export method filter, exported file details), `settings/default-deletion-intervals-for-job-types-read-only.cy.js` (C813000), `claiming/claiming-jobs-for-consecutive-runs-displayed-in-export-manager.cy.js`, MARC authority reporting specs.
- Fragments: `cypress/support/fragments/exportManager/` (exports.js, exportDetails.js, exportManagerSearchPane.js), `cypress/support/fragments/settings/export-manager/` (jobs.js, settings.js); API helper `cypress/support/api/export-manager.js`; constants `cypress/support/constants/export-manager/` (callouts, jobs, settings).

---

## Known Gaps / Items to Verify

- [ ] C813000 (and C784505 steps 2–3) name the 730-day settings rows "Orders (EDI)" / "Orders (csv)" and the subheader "Jobs deletion interval (days)"; code, Cypress constants, docs and UIEXPMGR-129 comment use "Claims" and "EDIFACT orders export" and en.json has "Job deletion intervals (days)". Use the code labels; the cases need a fix.
- [ ] C1263891 title says "User with all Export Manager permissions" but its preconditions grant View + DownloadAndResend (not the "manage"/All set). Download links depend on `Export manager: All`, not on type-specific capabilities (UIEXPMGR-132/135 Won't Do). UIEXPMGR-133/134/136/137 (type-specific download permissions for Bursar, Circulation log, Orders, Claims) are still Open.
- [ ] Job type column for CLAIMS jobs on the All tab: no `exportJob.details.type.CLAIMS` key, so the raw value "CLAIMS" is expected to show — verify in env.
- [ ] UI for manual "export now" of selected POLs (`/configs/{id}/execute`, MODEXPS-316) and effective-configs (MODEXPS-323) lives in ui-orders (UIOR-1556) — not in ui-export-manager; see `orders.md`.
- [ ] Email transmission (MODEXPW-625/642) and CSV ordering format (MODEXPW-628) have no TestRail cases in the Export Manager sections; "Sent to" for Email integrations is not handled specially in ExportEdiJobDetails.js (shows FTP fields) — verify.
- [ ] MODEXPW-640 (jobs Failed with "null (UnresolvedAddressException)" on snapshot) closed as not reproducible — environment issue, not a rule.
- [ ] TestRail cases updated after 2026-11-07 were not filtered separately; all titles in the four sections were reviewed.
