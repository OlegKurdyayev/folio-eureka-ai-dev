# FOLIO Course Reserves — Business Logic Context

> Built: 2026-07-21 · Refreshed: 2026-11-25 · Release: Umbrellaleaf R2 2026 (ui-courses master: v7.1.1 released for Trillium R1 2026, v7.1.2 in progress; mod-courses master: v1.4.14 released 2026-11-20, 1.4.15-SNAPSHOT) · Builder: build-app-context (full refresh, manual merge)

> Sources (2026-11-24 refresh): GitHub `folio-org/ui-courses` (translations/ui-courses/en.json + en_US.json, package.json, src/, CHANGELOG.md), `folio-org/mod-courses` (descriptors/ModuleDescriptor-template.json, ramls/*.json, src/main/resources/templates/db_scripts/schema.json, CourseAPI.java, CRUtil.java, NEWS.md), `folio-org/stripes-testing` (cypress/e2e, support/fragments/courses, support/dictionary), `folio-org/folio-integration-tests` (no Course Reserves Karate project), `folio-org/docs` (content/en/docs/Access/Courses/courses.md, Settings/Settings_courses/settings_courses.md); Jira UICR + MODCR (98 issues updated since 2024-06, ~35 read in detail: Ramsons R2 2024 → Trillium R1 2026 fixVersions); TestRail suite 21 section 1116 "Course Reserves" (subsections 1117 Courses — 8 cases, 1118 Reserve Items — 6, 1119 Settings — 3, 1125 Permissions — 1, 1126 "Courses : Read only" — 0 cases; 18 cases total, none created/updated since 2025-08).
> Exact UI texts below are from `translations/ui-courses/en.json` (key named) or a TestRail case (C-ID). `en_US.json` lags by one key (`editCourse.warning`, UICR-244) — otherwise identical.
> This area is legacy/lightly-maintained (most cases from 2020; dev by Index Data under limited funding — UICR-234/UICR-237 comments) — real cases are thinner than newer areas; still apply this skill's full verification-depth rules, don't mirror legacy shallowness.

---

## What is Course Reserves

The Courses app (route `/cr`) lets library staff link Inventory items to an academic course so students can find and borrow "reserve" materials tied to that course/term. A **Course** belongs to a **Course listing** (term, course type, registrar ID, external ID, location, instructors, reserves); several courses sharing one course listing are **cross-listed** and share instructors, listing info and reserve items (docs courses.md). Reserve items get a **temporary location** (from the course listing's Location) and optionally a **temporary loan type** written to the Inventory item, and can be tracked for processing/copyright status. Patron discovery of courses needs an external discovery layer / LTI (docs courses.md; Confluence FOLIOtips "Course Reserves - LTI connectivity").

---

## Key Terms

| Term | Definition |
|---|---|
| Course | Organizing record: name, description, department, course code (`courseNumber`), section, number of students; required `name`, `departmentId`, `courseListingId` (mod-courses `ramls/course.json`) |
| Course listing | Shared container: term (required), course type, registrar ID, external ID, location, service point, instructors, reserves (`ramls/courselisting.json`). Not visible as its own record in the UI |
| Cross-listed course | Another course on the same course listing ("Crosslist" action) — shares reserve items, instructors and listing info (docs courses.md "Adding a cross-listed course") |
| Reserve item | An Inventory item added to a course listing by barcode (or Fast Add); stores a `copiedItem` snapshot of Inventory data (`ramls/reserve.json`) |
| Term | Configured name + start/end date (all required, `ramls/term.json`); course Status is derived from it |
| Course type / Department / Processing status / Copyright status / Role | Settings vocabularies, `name` required, `description` optional (`ramls/coursetype.json`, `department.json`, `processingstatus.json`, `copyrightstatus.json`, `role.json`) |
| Instructor | Person on a course listing: `name` required; optional `userId` (linked FOLIO user), `barcode`, `patronGroup` (`ramls/instructor.json`) |

---

## Record Hierarchy / Architecture

```
Term ──< Course listing (termId FK, courseTypeId FK, locationId, servicepointId)
           ├──< Course (departmentId FK)   — 1..n; >1 = cross-listed
           ├──< Instructor (userId optional)
           └──< Reserve (itemId, copiedItem{...}, processingStatusId FK,
                         copyrightTracking.copyrightStatusId FK, temporaryLoanTypeId,
                         startDate/endDate)
                   └── writes temporaryLocationId / temporaryLoanTypeId onto Inventory item
```
FKs from `mod-courses/src/main/resources/templates/db_scripts/schema.json`; no unique indexes (duplicate-item check is in code, see rule 2).

---

## Lifecycle / Status

### Course status (computed in UI, not stored)
`Pending` (`status.pending`) / `Active` (`status.active`) / `Inactive` (`status.inactive`). Calculated client-side from the course listing's Term dates vs. today: today < term start → Pending; term end < today → Inactive; otherwise Active; no term object → empty value (ui-courses `src/components/Courses.js` `calculateStatus`). Server has no status field (MODCR-104, open) — hence Status is not sortable/searchable (UICR-179, blocked).

### Reserve lifecycle (side effects on Inventory item)
| Event | Effect on Inventory item | Source |
|---|---|---|
| Add reserve (POST) and course listing has Location | item `temporaryLocationId` := course listing location | CourseAPI.java `handleWriteReserves`; MODCR-107; docs courses.md |
| Add reserve, listing has no Location | item temp location left as is; copied into reserve | CourseAPI.java / CRUtil.java |
| Add reserve | reserve `temporaryLoanTypeId` := item's existing temp loan type if the reserve has none; item temp loan type written only if reserve carries one | CRUtil.java (copy), CourseAPI.java |
| Edit reserve (PUT) | item `temporaryLocationId` := reserve's temp location (null clears it); temp loan type written if set | CourseAPI.java; docs courses.md "Editing a reserve item" |
| Remove reserve (DELETE) | item `temporaryLocationId` set to null; **temporary loan type NOT cleared** | CourseAPI.java `deleteReserve`/`resetItemTemporaryLocation`; MODCR-138 (draft enhancement); docs courses.md |
| Item changed in Inventory after reserving (title, call number, move holdings/item, URI) | reserve `copiedItem` NOT updated — remove and re-add to refresh | UICR-125 (Won't do), MODCR-150 (open), docs courses.md |

---

## UI structure [from source — ui-courses src, v7.1.x]

- **Search & filter pane** with segment toggle `Courses` / `Reserves` (`filters.courses`, `filters.reserves`).
  - Courses segment: search box (`searchInputLabel` "Search courses") with scope `All fields`, `Course name`, `Course code`, `Section`, `Instructor`, `Registrar ID`, `External ID` (`index.courses.*`); filters `Department`, `Course type`, `Term`, `Location` (`filters.*`). Trailing asterisks are collapsed before the query (UICR-212); filter option lists match anywhere in the value, not left-anchored (UICR-160).
  - Reserves segment: scope `All fields`, `Title`, `Barcode`, `Call Number` (`index.reserves.*`); filters `Processing status`, `Copyright status`, `Permanent location`, `Temporary location`, `Term` (ReservesSearchPane.js; docs courses.md).
- **Courses results list** columns: `Course name` (hyperlink, not hideable), `Course code`, `Section`, `Registrar ID`, `Department`, `Start date`, `End date`, `Instructor` (names joined with "; "), `Status` (Courses.js). Sortable only by Course name, Course code, Section; default sort Course name (Courses.js `nonInteractiveHeaders`; UICR-179; UICR-225). Column chooser via Actions > Show columns (UICR-132). Actions menu: `New` (only with `course-reserves-storage.courses.item.post`).
- **Reserves results list** columns: `Title` (link to Edit reserve only with `reserves.item.put`), `Barcode`, `Processing status`, `Permanent location`, `Temporary location` (Reserves.js).
- **Course detail pane** accordions: `Course data` (`accordion.data`), `Cross-listed courses`, `Instructors` (card header `{count} instructor(s)`), `Term`, `Organization`, reserves accordion titled `{count} item(s)` (`accordion.reserves`), `Developer info`, `Notes`. Actions menu (shown only with `course-reserves-storage.courses.item.put`): `Edit`, `Crosslist`, `Duplicate`, `Delete` (Delete needs `courses.item.delete` and is **disabled while the course has reserves**) (Course.js; UICR-201).
- **Reserve row** (detail pane): item title link opens Inventory item in a new tab (`/inventory/view/{instance}/{holdings}/{item}`), Barcode, Contributor, Permanent location, Temporary location, Temporary loan type, Call number, Volume, Copy, Enumeration, Status (item status, live from Inventory), Processing status, Start/End date (falls back to the Term dates if the reserve has none), URL/PDF link, copyright fields. Edit (pencil, `reserves.item.put`) / Remove (trash, `reserves.item.delete`) icons (ViewCourseReserves.js).
- **Add item**: `Enter or scan barcode` + `Add item`; `Add Fast Add item` button (needs the create-inventory-records plugin + "Fast add: Create") (AddReserve.js, AddFastAddReserve.js, docs).
- **Create/Edit course form**: accordions `Basic course information` (Course name*, Department*, Course code, Section, Number of students, Course description) and `Course listing information` (Course type, Registrar ID, External ID, Term*, Location) — on Edit the listing section shows a dismissible warning banner (`editCourse.warning`). Crosslist form: `New course within listing` with `Cross listing information` pre-filled (CourseFormInfo.js, CourseFormListing.js).
- **Edit reserve form** (`Edit reserve`): Title + contributors at top, `Temporary location`, `Temporary loan type`, `Processing status`, `Start Date`, `End Date`; `Copyright Information` (`Copyright tracking needed?`, `Additional sections of this item used`, `Copyright status`, `Total number of pages in item`, `Total number of pages used`, `Total % of pages used`, `Payment based on`); card `Courses to which this item is reserved` (ReserveForm sections; C9184).
- **Instructor form**: `Name`* , `Barcode`, `Look up user` (find-user plugin fills name/barcode, sets userId) (InstructorForm.js).
- **Duplicate modal** (`Duplicate course`): required `Term` select (options sorted by name), `Duplicate all cross-listed courses` checkbox (default from Display settings), button `Create duplicate course(s)` (DuplicateCourseModal.js).
- **Settings > Courses**: `Terms`, `Course Types`, `Course Departments`, `Processing Statuses`, `Copyright Statuses`, `Roles`, `Display settings` (`Duplicate cross-listed courses`) (settings/index.js, en.json).
- App context menu: `Courses app search`, `Keyboard shortcuts` (UICR-166, UICR-124). Shortcuts registered: new, edit, save, cancel, duplicateRecord, expandAllSections, collapseAllSections (src grep).

### Key navigation paths
`Courses > Courses segment > course > Actions > Edit/Crosslist/Duplicate/Delete` · `Courses > course > reserves accordion > pencil/trash` · `Courses > Reserves segment > Title link > Edit reserve` · `Settings > Courses > Terms | Course Types | Course Departments | Processing Statuses | Copyright Statuses | Roles | Display settings`

---

## Exact UI Texts [en.json keys]

### Toasts / callouts
| Event | Message | Key |
|---|---|---|
| Item added to reserves | `Added item "{title}"` | `addItem.addedItem` |
| Duplicate barcode added | `Duplicate barcode {barcode}: this item is already on reserve for this course` | `addItem.duplicateItem` |
| Add item failed (e.g. unknown barcode → BE `No item found for barcode {barcode}`) | `Failed to add item {barcode}: {message}` | `addItem.failure` |
| Add item with empty barcode | `Please enter a barcode before selecting "Add item"` | `addItem.missingBarcode` |
| Remove reserve failed | `Remove reserve failed: {message}` | `removeReserve.failure` |
| Remove instructor failed | `Remove instructor failed: {message}` | `removeInstructor.failure` |
| Course deleted (green) | `The course <b>{name}</b> with course number <b>{number}</b> was deleted.` | `courseDeleted` (UICR-207) |
| Term deleted | `The term <b>{term}</b> was successfully <b>deleted</b>` | `cv.termDeleted` |

### Modals / banners
| Modal / banner | Text | Key |
|---|---|---|
| Delete course modal title | `Confirm deletion of course` (buttons `Delete` / `Cancel`) | `reallyDelete`, `button.delete`, `button.cancel` |
| Delete term modal title / body | `Delete term` / `The term <b>{term}</b> will be <b>deleted.</b>` | `cv.deleteEntry`, `cv.termWillBeDeleted` |
| Term in use | Header `Cannot delete the term`; body `This term cannot be deleted, as it is in use by one or more records.` | `cv.cannotDeleteTermHeader`, `cv.cannotDeleteTermMessage` |
| Edit course listing warning | `Warning: changes made in this section will also affect all cross-listed courses.` | `editCourse.warning` (en.json only; UICR-244) |
| Duplicate modal | title `Duplicate course`; checkbox `Duplicate all cross-listed courses`; button `Create duplicate course(s)`; new name suffix `Duplicate` | `duplicateCourse`, `duplicateCourse.duplicateCrosslistedCourses`, `button.createDuplicateCourses`, `duplicateCourse.duplicatedCourseIndicator` |
| Display setting help | `If this setting is checked, when duplicating a course the "Duplicate cross-listed courses" checkbox in the Duplicate modal will be marked as checked by default.` | `settings.display.duplicateCrosslistedCourses.description` |
| Copyright view | `No copyright tracking required` / `This item requires copyright tracking:` | `copyrightTracking.no` / `.yes` |

### Validation messages
- Number of students not an integer: `This value must be an integer` (`errors.mustBeInteger`).
- Missing Course name / Department / Term: stripes-core `missingRequiredField` (text not in ui-courses translations — take from env).

### Button / pane labels
`New course` (`create`), `Create course` (`createCourse`), `Edit course`, `Crosslist` / `Crosslist course`, `Duplicate`, `Delete`, `Edit reserve`, `Remove`, `Add instructor`, `Edit instructor`, `Remove instructor`, `Look up user`, `Add item`, `Add Fast Add item`, `Add existing Inventory item as a reserve`, `Create new Inventory item and add it as a reserve`.

### Field labels
Course: `Course name`, `Course description`, `Department`, `Course code` (`field.number`), `Section`, `Number of students`, `Course type`, `Registrar ID`, `External ID`, `Term`, `Start date`, `End date`, `Location`. Instructor: `Name`, `Barcode`, `Patron group`. Reserve: `Item barcode`, `Contributor`, `Permanent location`, `Call number`, `Volume`, `Copy`, `Enumeration`, `Status`, `Temporary location`, `Temporary loan type`, `Processing status`, `URL/PDF link`, plus copyright fields above. Note: `field.serviceDesk` / `column.serviceDesk` ("Service desk") exist in en.json but are not rendered by any v7.1.x form or results column (src grep).

### Empty-search message pattern (C9185)
`No results found for "{search term}". Please check your spelling and filters.`

---

## API surface (mod-courses ModuleDescriptor-template.json)

Interface `course-reserves-storage` 0.3 (UI requires 0.2) + `term-storage`, `department-storage`, `course-type-storage`, `processing-status-storage`, `copyright-status-storage`, `role-storage`, `course-storage`, `reserves-storage` 0.1.

| Endpoint | Methods | permissionsRequired (pattern) |
|---|---|---|
| `/coursereserves/courselistings[/{id}]` | GET, POST, PUT, DELETE | `course-reserves-storage.courselistings.{collection.get,item.post,item.get,item.put,item.delete}` |
| `/coursereserves/courselistings/{id}/courses[/{c_id}]` | GET, POST, PUT, DELETE | `…courselistings.courses.*`; POST is `course-reserves-storage.courselisting.courses.item.post` (singular "courselisting") |
| `/coursereserves/courselistings/{id}/instructors[/{i_id}]` | GET, POST, PUT, DELETE | `…courselistings.instructors.*` |
| `/coursereserves/courselistings/{id}/reserves[/{r_id}]` | GET, POST, PUT, DELETE | `…courselistings.reserves.*` |
| `/coursereserves/courses[/{id}]`, `/coursereserves/reserves[/{id}]` | GET, POST, PUT, DELETE | `…courses.*`, `…reserves.*` (search endpoints used by both segments) |
| `/coursereserves/{terms,departments,coursetypes,processingstatuses,copyrightstatuses,roles}[/{id}]` | GET, POST, PUT, DELETE | `…terms.*`, `…departments.*`, `…course-types.*`, `…processing-statuses.*`, `…copyright-statuses.*`, `…roles.*` |

Behaviour notes (CourseAPI.java / CRUtil.java):
- POST reserve accepts `itemId` or `copiedItem.barcode`; unknown barcode → 400 `No item found for barcode …`; same item twice on one listing → 422 `itemId {id} is already in use for courseListing {id}`.
- POST course under a listing with mismatched `courseListingId` → 422 `listingId should be …`.
- DELETE course listing deletes its instructors first (MODCR-121); deleting a referenced term/course type/department/processing or copyright status fails on FK (`… is still referenced from table …`, UICR-203).
- POST instructor with `userId` → barcode and patron group overwritten from the user record (`users` interface).
- Reserve start/end dates are stored as UTC from local date (UICR-145, UICR-228). Display settings live in mod-configuration: `configurations/entries?query=(module=COURSES and configName=display)` (util/manifest.js).
- `copiedItem.callNumber` = prefix, call number, suffix joined with spaces (MODCR-143; `makeCallNumber`).

---

## Capability Sets (Eureka) [docs courses.md / settings_courses.md + ui-courses package.json]

Display names: ui-courses `package.json` stripes.permissionSets and `translations/ui-courses/en.json` `permission.*`. Eureka names: docs.folio.org `Access/Courses/courses.md` "Permissions" table (Resource / Type / Action), spelled in the skill form `Type - Resource - Action`. Only `Data - UI-Courses - Manage` is additionally confirmed in stripes-testing (`capabilitySets.uiCourses`); the rest rely on docs alone, not a live env.

| Okapi display name (package.json / en.json `permission.*`) | Permission | Eureka capability set (docs courses.md) |
|---|---|---|
| `Courses: All permissions` | `ui-courses.all` (includes maintain-settings) | `Data - UI-Courses - Manage` (also stripes-testing `capabilitySets.uiCourses`) |
| `Courses: Read all` | `ui-courses.read-all` | `Data - UI-Courses - View` |
| `Courses: Read, add, and edit courses` | `ui-courses.read-add-edit` (+ instructors CRUD, no course delete) | `Data - UI-Courses Courses - Manage` |
| `Courses: Read, add, edit, and delete courses` | `ui-courses.maintain-courses` | (capability set name not confirmed — verify): absent from docs table, lost in Eureka mapping (UICR-239, open) |
| `Courses: Add and edit courses' reserved items` (docs spell it "a courses’ reserved items") | `ui-courses.add-edit-items` (no remove; includes inventory item put, loan types, `users.collection.get` — UICR-235) | `Data - UI-Courses Add-Edit-Items - Manage` |
| `Courses: Add, edit, and remove courses' reserved items` | `ui-courses.maintain-items` | `Data - UI-Courses Items - Manage` |
| `Settings (Courses): Can view course settings` | `ui-courses.view-settings` | `Settings - UI-Courses Settings View - View` (docs settings_courses.md) |
| `Settings (Courses): Can create, edit and delete course settings` | `ui-courses.maintain-settings` | `Data - UI-Courses Settings - Manage` (docs settings_courses.md) |
| `Fast add: Create` | plugin-create-inventory-records | `Data - UI-Plugin-Create-Inventory-Records - Create` |

Notes need separate Notes capabilities (docs; C421983 preconditions). TestRail C9185/C9186/C605979 preconditions list the Okapi display names above (incl. `Courses: Read, add, edit, and delete courses`). For new cases use the Eureka names above and confirm in env.

---|---|---|
| `Courses: All permissions` | `ui-courses.all` (includes maintain-settings) | UI-Courses · Data · Manage |
| `Courses: Read all` | `ui-courses.read-all` | UI-Courses · Data · View |
| `Courses: Read, add, and edit courses` | `ui-courses.read-add-edit` (+ instructors CRUD, no course delete) | UI-Courses Courses · Data · Manage |
| `Courses: Read, add, edit, and delete courses` | `ui-courses.maintain-courses` | **not in docs table** — lost in Eureka mapping (UICR-239, open) |
| `Courses: Add and edit courses' reserved items` | `ui-courses.add-edit-items` (no remove; includes inventory item put, loan types, `users.collection.get` — UICR-235) | UI-Courses Add-Edit-Items · Data · Manage |
| `Courses: Add, edit, and remove courses' reserved items` | `ui-courses.maintain-items` | UI-Courses Items · Data · Manage |
| `Settings (Courses): Can view course settings` | `ui-courses.view-settings` | UI-Courses Settings View · Settings · View |
| `Settings (Courses): Can create, edit and delete course settings` | `ui-courses.maintain-settings` | UI-Courses Settings · Data · Manage |
| `Fast add: Create` | plugin-create-inventory-records | UI-Plugin-Create-Inventory-Records · Data · Create |

stripes-testing uses `capabilitySets.uiCourses` = Data / `UI-Courses` / Manage (cypress/support/dictionary/capabilitySets.js). Notes need separate Notes capabilities (docs; C421983 preconditions). Legacy display names may be used in Okapi-era preconditions; for new cases use the Eureka resource names above and confirm in env.

---

## ECS notes
No consortium-specific behaviour exists in ui-courses or mod-courses (Jira UICR/MODCR text search "consortium"/"ECS": none; no `ecs` code paths). Course Reserves data is tenant-local; in an ECS member tenant the app reads that tenant's items via `inventory.items`. TestRail cases all have `custom_ecs_enabled` false/null. Treat ECS as untested — Known Gap.

---

## Existing automation coverage
- Karate (folio-integration-tests): **none** — no Course Reserves project/feature (repo file list, 2026-11-24).
- Cypress (stripes-testing): `cypress/e2e/courses/create-edit-note.cy.js` (C421983, notes on a course); `cypress/e2e/notes/notes-actions-view-permission.cy.js` (uses course API helpers); `cypress/e2e/eureka/login/login-url-redirect.cy.js` (C451519, opens `/cr` with `UI-Courses`). Fragments: `cypress/support/fragments/courses/courses.js` (API helpers for courses, listings, terms, departments), `coursesView.js` (notes accordion). No Cypress coverage for reserves, instructors, crosslisting, duplicate, settings.
- TestRail: all 18 cases `custom_automation_type` = 2 except the Cypress-backed C421983.

---

## Common Verification Patterns

### Create a course (C9171 — partly outdated)
Preconditions: Term and Department configured; user with course-create capability.
1. Courses segment > Actions > `New` → `Create course` form.
2. Fill required `Course name`, `Department`, `Term` (CourseFormInfo.js/CourseFormListing.js, UICR-93) → `Save & close`.
3. Expected: course appears in the list (default sorted by Course name — UICR-225) and opens in the detail pane.
> C9171 step 2 says "Course Name, Term, and Service Desk" — no Service desk field exists in v7.1.x and Department is required (see Known Gaps).

### Add a reserve item and cross-check Inventory (C9183)
1. On the course, enter the barcode in `Enter or scan barcode`, click `Add item`.
2. Expected: toast `Added item "{title}"`; row appears (reserves sorted by title — CourseRoute.js `sortby copiedItem.title`).
3. Compare with Inventory item: Title, Contributors, effective Call number (prefix/number/suffix with spaces — MODCR-143), Permanent location, Status, Copy, Volume.
4. Inventory item temporary location = course Location (if set) (MODCR-107); reserve Start/End date = Term dates (docs).

### Remove a reserve item — verify Inventory side effects (C9190)
1. Note temporary location and temporary loan type on the Inventory item.
2. Click trash on the reserve row.
3. Expected: row gone; Inventory item temp location cleared; **temp loan type still present** (docs courses.md; MODCR-138).

### Instructors — two add paths
- Free text: `Add instructor` → `Name` (required) [+ `Barcode`] → no user link (C9173).
- `Look up user` → select user → name + barcode filled; patron group stored from the user record (C9187; CourseAPI.java).
- Docs: an instructor with a FOLIO user record cannot be edited (docs courses.md) — verify in env. Remove (C9189): name, barcode and patron group disappear.

### Crosslisting (C9174)
`Crosslist` → `New course within listing` with `Cross listing information` pre-filled → fill `Course name` (+ required Department) → `Save & close` → new course in `Cross-listed courses` on the original and in the main list; reserves/instructors shared.

### Duplicate (C343206 + DuplicateCourseModal.js)
Pick Term (required) → `Create duplicate course(s)` → new course named `{name} - Duplicate` opens; copies course data, course type, registrar/external ID, location, service point, instructors (name, userId), reserves (by itemId); cross-listed courses duplicated only if checkbox checked; checkbox default = Display setting.

### Delete course (UICR-201/206/207)
Actions > `Delete` disabled while reserves exist → remove reserves → `Delete` → `Confirm deletion of course` → `Delete` → green toast `courseDeleted`; if no cross-listed courses remain the course listing (and its instructors) is deleted too, so its Term becomes deletable.

### Search & filter (C9185, C9186)
Courses scopes: All fields, Course name, Course code, Section, Instructor, Registrar ID, External ID; filters Department, Course type, Term, Location (alone + combined). Reserves segment: scopes All fields/Title/Barcode/Call Number; filters Processing status, Copyright status, Permanent location, Temporary location, Term. Verify one hit and one no-hit per scope.

### Settings (C9175, C9176, UICR-195)
Each vocabulary: `New` → Name (+ Description; Term adds `Start Date`/`End Date`) → Save → immediately selectable in course/reserve forms. With view-only capability the `New` button is hidden (UICR-195). Deleting an in-use entry is blocked (FK).

### Notes accordion (C421983)
`New note`/`Edit note` from a course do NOT show `Check out app` / `Users app` checkboxes (regression guard; UICR-189).

### Edit Reserve item form (C9184) / Temporary loan type by name (C357543)
Form shows Title + Contributors at top; `Temporary loan type` always renders the name, never a UUID (UICR-157 perms fix).

---

## Key Business Rules for Test Cases

1. Adding a reserve sets the Inventory item's temporary location to the course listing Location (when one is set); removing the reserve clears the item's temporary location — always verify in Inventory, not only in Courses. (C9190; mod-courses CourseAPI.java; MODCR-107)
2. The same item cannot be added twice to the same course listing: BE returns 422 and the UI shows `addItem.duplicateItem`, not a silent no-op. (en.json `addItem.duplicateItem`; CourseAPI.java `checkUniqueReserveForListing`)
3. Reserve fields Title, Contributors, Call number, Permanent location, Copy, Volume come from the Inventory snapshot at add time and must match the item; Status is read live. (C9183; CRUtil.java; ViewCourseReserves.js)
4. Instructors can be added as free text (Name required, Barcode optional) or via `Look up user` (user link; barcode + patron group taken from the user record on save). (C9173, C9187; InstructorForm.js; CourseAPI.java)
5. Crosslisting pre-fills `Cross listing information` from the source course and leaves `Basic course information` blank; cross-listed courses share reserves, instructors and listing info. (C9174; docs courses.md)
6. The Duplicate modal's `Duplicate all cross-listed courses` default comes from Settings > Courses > Display settings; set and verify that setting first. (C343206; en.json `settings.display.duplicateCrosslistedCourses.description`)
7. A Term, Course type, Department, Processing status or Copyright status referenced by a record cannot be deleted (FK); only Term has a custom message (`cv.cannotDeleteTermMessage`). (schema.json foreignKeys; UICR-203; en.json)
8. A read-only user (`Courses: Read all`) must not see New/Edit/Crosslist/Duplicate/Delete, reserve edit/remove icons or instructor add/edit/remove — check absence of controls, not only failed saves. (Section 1126; Course.js, ViewCourseReserves.js, ViewCourseInstructors.js `hasPerm`)
9. Courses list sorts by Course name (default, A–Z with locale-aware order, e.g. A–Å), Course code and Section, reversing on second click; it must also sort when no query is entered. (C825318; UICR-225)
10. Item Status is fetched for at most the first 45 reserves of a course (limit lowered from 100 to keep URL under 2 KB); a course with >80 items must open without the 414 / "operation GET on resource 'items' failed" error. (UICR-233; CourseRoute.js `MAX_RESERVES_TO_REQUEST`; C605979 / UICR-210 superseded)
11. Department, Course type, Term and Location filters work alone and combined; Course type filter queries `courseListing.courseTypeId`. (C9186; CoursesRoute.js)
12. New Settings vocabulary entries become immediately selectable in the course / reserve forms — no activation step. (C9175, C9176)
13. The Course type filter carried a known-issue note (BF-809) in C9186; current code implements it like the other filters — re-verify rather than expecting failure. (C9186; CoursesRoute.js)
14. Required course fields are Course name, Department and Term (front-end validation); Number of students accepts integers only (`errors.mustBeInteger`). (CourseFormInfo.js, CourseFormListing.js; UICR-93; course.json, courselisting.json required)
15. Course Status is computed from the Term: Pending before start, Active between start and end, Inactive after end; not sortable. (Courses.js `calculateStatus`; MODCR-104; UICR-179)
16. Actions > Delete is disabled while the course has reserve items; after confirmation a green toast `courseDeleted` appears. (Course.js `disabled={hasReserves}`; UICR-201; UICR-207; docs courses.md)
17. Deleting the last course on a course listing also deletes the listing and its instructors, so the Term becomes deletable; deleting a cross-listed course leaves the listing and its reserves to the remaining courses. (DeleteCourseModal.js; UICR-206; MODCR-121; docs courses.md)
18. Removing a reserve does NOT clear the item's temporary loan type in Inventory (design limitation, enhancement MODCR-138 in draft). (CourseAPI.java `resetItemTemporaryLocation`; docs courses.md)
19. Editing a reserve's Temporary location / Temporary loan type writes the value to the Inventory item on save; clearing temporary location on edit clears it on the item. (docs courses.md; CourseAPI.java PUT path)
20. Inventory changes after reserving (title, call number, move holdings/item, URI) are not propagated to the reserve; workaround is remove + re-add. (UICR-125 Won't do; MODCR-150; docs courses.md)
21. Reserve Start/End dates default to the Term dates; edited dates display in the tenant time zone with no day shift (test with e.g. America/Vancouver). (ViewCourseReserves.js; UICR-205; UICR-228)
22. Duplicating a course names the copy `{name} - Duplicate`, requires a Term, and copies instructors and reserves; known limitation — reserves' temporary locations are reset to the course Location on both courses (workaround: clear course Location before duplicating). (DuplicateCourseModal.js; docs courses.md; UICR-234 Won't do; UICR-237 open)
23. Editing Course listing information shows the warning banner `editCourse.warning`, since changes affect all cross-listed courses; the string is translatable (Lokalise). (en.json; UICR-244)
24. With only `Settings (Courses): Can view course settings` the `New` button is hidden on all Settings > Courses vocabulary pages. (UICR-195; CHANGELOG 7.1.0)
25. Search terms with multiple trailing asterisks are reduced to one before querying, so `carla**` behaves like `carla*` without a CQL error. (UICR-212; CoursesRoute.js/ReservesRoute.js)
26. Adding with an empty barcode shows `addItem.missingBarcode`; an unknown barcode shows `addItem.failure` with the BE message. (AddReserve.js; CRUtil.java)
27. A URL/PDF link without `http(s)://` must open as an external link, not be treated as a course ID. (UICR-184; CHANGELOG 7.1.0)
28. Item status and URL/PDF link must still show after a full page refresh of a course. (UICR-221)
29. Copied call number shows spaces between prefix, call number and suffix (e.g. `TEST MCN FICTION`). (MODCR-143; UICR-236; CRUtil.java `makeCallNumber`)
30. Reserve-item add/edit/remove controls depend on `course-reserves-storage.reserves.item.{post,put,delete}`: `Courses: Add and edit courses' reserved items` shows add/edit but no Remove icon. (ViewCourseReserves.js; package.json; docs courses.md)

---

## Authoring style (measured 2026-07-23)

Course Reserves: **`Other` ~61%** / Func ~39%, median **~3 steps** (short), `User Journey` ~0%. Compact atomic cases — create/edit a course, add/remove a reserve item, assign an instructor, crosslist, verify a display field. `Other` for display/config; `Functional` for behavior that changes item temporary loan type / location on reserve add-remove. Setup (a course, an instructor, an item) lives in Preconditions; steps stay short.

---

## Edge cases from bugs (for regression checks)
- >45 reserves: statuses only for first 45; no 414 error (UICR-233). >20 reserves: status and temp location must render (UICR-210, UICR-192).
- Tenant time zone ≠ UTC: reserve dates after edit (UICR-205, UICR-228).
- Delete term after deleting all its courses — orphan listings no longer block it (UICR-203, UICR-206).
- Sorting with empty query (UICR-225); multiple asterisks (UICR-212).
- Instructors removed from the list update the display without reload (UICR-177).
- Non-English locale: edit-course warning translated (UICR-244); term settings messages refer to terms, not patron groups (UICR-229).
- Malformed URL/PDF link (UICR-184); page refresh losing item status/link (UICR-221).

---

## Known Gaps / Items to Verify

- [ ] **Course create required fields conflict**: C9171 lists "Course Name, Term, and Service Desk"; docs courses.md says only Course Name + Term; code (CourseFormInfo.js, UICR-93) and `course.json` require Department as well, and there is no Service desk field in v7.1.x. Update C9171 when touched.
- [ ] **`Courses: Read, add, edit, and delete courses` in Eureka** — missing from the docs capability table; UICR-239 (open) says it was lost in the Okapi→Eureka mapping and deleting courses currently needs `Courses: All permissions` (which also grants settings). Confirm capability set names in env.
- [ ] Eureka capability names come from docs (resource/type/action), not from a live env; only `Data - UI-Courses - Manage` is also in stripes-testing — verify exact display in Settings > Authorization roles. Docs spell the add-edit-items display name "Add and edit a courses’ reserved items" vs `Courses: Add and edit courses' reserved items` in package.json/en.json — verify in env.
- [ ] Course type filter (BF-809 note in C9186): no Jira record of a fix; code implements it — re-verify and update C9186.
- [ ] C605979 expects status for "all 20 or more" items; since UICR-233 only the first 45 get status — case is valid up to 45 items only.
- [ ] Temporary loan type "inherited from the course" (docs courses.md) — there is no course-level loan type field; reserves only inherit the item's own temp loan type or one set on Edit reserve. Clarify wording.
- [ ] Docs say a user-linked instructor cannot be edited; InstructorForm.js has no explicit lock — verify in env.
- [ ] Service point (`servicepointId`) is in the course listing schema and copied on duplicate but not editable in the UI — how it gets set is unknown.
- [ ] ECS behaviour untested — no ECS cases or code.
- [ ] No Karate coverage; Cypress covers only notes. Candidates for API automation: reserve add/remove side effects on item temp location/loan type, duplicate-item 422, course-listing cascade delete, FK delete blocks.
- [ ] Display settings stored in mod-configuration (`configurations/entries`, module COURSES) — check after mod-configuration deprecation; `ui-courses.view-settings` includes `configuration.entries.collection.get`.
- [ ] Pending migrations: mod-locations APIs (UICR-243, MODCR-149 drafts); reserve copy vs. live URI (UICR-115 / MODCR-150 open).
- [ ] Keyboard shortcut table (C350570) only as an attachment image; registered shortcut names listed above, key combos not captured.
- [ ] Section 1126 "Courses : Read only" has 0 cases — read-only rule 8 has no TestRail case yet.
- [ ] C9185 (search courses) and C9186 (course filters) sit in section 1118 "Reserve Items" but test the Courses segment; the Reserves segment search/filters have no TestRail case (bodies read 2026-11-25).
- [ ] `field.serviceDesk` / `column.serviceDesk` translations unused in v7.1.x UI — do not assert a "Service desk" column/field.
- [x] Copyright/Processing status settings behaviour — confirmed (C9176).
- [x] Notes accordion — regression guard, not a bug (C421983; UICR-189).
- [x] Settings entities beyond Terms — confirmed (C9175, C9176).

---

## Change log
- 2026-11-24 full refresh: merged the 2026-11-07 delta (no Course Reserves changes; its unrelated STCOR/MODROLESKC/MDEXP/ERM bullets dropped). Corrected the results-list columns (no Service desk column; Registrar ID added) and the create-course required fields; added Reserves search segment, API surface, Eureka capabilities, inventory side-effect table, coverage, edge cases, rules 14–30. UI strings changed since 2026-07-21: added `editCourse.warning` (UICR-244); none removed.
- 2026-11-25 audit remediation: Release header normalized; capability table recast to `Type - Resource - Action` with sources (maintain-courses marked not confirmed); vocabulary/cross-listed rows tagged; read TestRail section 1118 bodies (C9183, C9190, C9184, C9185, C9186, C605979), added misfiled-case gap.
