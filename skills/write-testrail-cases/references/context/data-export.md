# FOLIO Data Export App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Data Export app before generating test cases.
> Sources: https://docs.folio.org/docs/metadata/data-export/ and https://docs.folio.org/docs/settings/settings_data_export/

---

## What is Data Export

The Data Export app exports bibliographic, holdings, item, and authority data from FOLIO as MARC 21 files (.mrc). It accepts record identifiers (UUIDs) or CQL queries to define the record set, applies a job profile to determine the output format, and generates a downloadable MARC file.

Exports can be triggered from three places:
1. **Data Export app** — by uploading a .csv (UUIDs) or .cql (CQL query) file
2. **Inventory app** — quick export via Actions → Export instances (MARC)
3. **MARC Authority app** — quick export via Actions → Export selected records (CSV/MARC)

---

## What Can Be Exported

| Record type | Format | Source |
|---|---|---|
| Instance (with underlying MARC) | MARC 21 (.mrc) | SRS — original MARC record used |
| Instance (FOLIO-native, no MARC) | MARC 21 (.mrc) | Generated on-the-fly from Inventory data |
| Holdings | MARC 21 (.mrc) | Generated from Inventory Holdings fields |
| Items | MARC 21 (.mrc) | Generated from Inventory Item fields |
| Authority records (not deleted) | MARC 21 (.mrc) | SRS — uses Default authority export job profile |
| Authority records (set for deletion) | MARC 21 (.mrc) | SRS — uses Deleted authority export job profile only |
| UUIDs only | .csv | Any record type; downloaded immediately |

---

## Profile Hierarchy (2 levels)

```
Job Profile  (entry point — what to run)
  └── Field Mapping Profile  (what fields to include and how to transform them)
```

### Job Profile
- Links a record type to a field mapping profile
- Required field: Name
- Required field: Mapping profile (select from existing)
- Optional: Description
- **Once a job profile has been used for an export job: cannot be edited or deleted**
- Can be duplicated

### Field Mapping Profile
- Determines which FOLIO fields map to which MARC fields in the output
- Required fields: Name, Output format (MARC only), at least one FOLIO record type
- FOLIO record types (checkboxes; at least one required):
  - **Source record storage (entire record)** — exports full underlying MARC from SRS
  - **Inventory instance (selected fields)** — exports selected Instance fields, generated on-the-fly
  - **Holdings** — appends Holdings data to MARC output
  - **Item** — appends Item data to MARC output
- Optional: Description
- Optional: **Fields suppression** — exclude specific MARC fields from export
- Optional: **Transformations** — map Inventory/Holdings/Item fields to specific MARC field/indicator/subfield combinations
- **While a field mapping profile is assigned to a job profile: cannot be edited or deleted**
- Can be duplicated

### Relationship rule
- Field mapping profile → assigned to job profile → used in export
- Must unassign (or delete) the job profile before the field mapping profile can be edited/deleted

---

## Default Job Profiles (System-Provided)

FOLIO ships with default profiles. Typical defaults:

| Profile name | Purpose |
|---|---|
| **Default instance export job profile** | Exports instances; uses SRS MARC for MARC instances; generates simplified MARC for FOLIO-native instances |
| **Default authority export job profile** | Exports non-deleted authority records from SRS |
| **Deleted authority export job profile** | Exports authority records set for deletion (LDR 05 = 'd') within the purge window |
| **Default holdings export job profile** | Exports holdings data |

### Authority export rules (critical for test cases)

| Job profile | Exports | Rejects with error |
|---|---|---|
| Default authority export | Non-deleted authority records | Records set for deletion → *"Authority record \<UUID\> is set for deletion and cannot be exported using this profile"* |
| Deleted authority export | Records set for deletion (within 7-day purge window) | Non-deleted records → *"This profile can only be used to export authority records set for deletion"* |
| Either profile | Records purged (past 7-day window) | → *"Record not found"* |

---

## Export Workflows

### Method 1: Upload UUIDs from Data Export app
1. Prepare a .csv file with one UUID per row
2. Jobs pane → drag-and-drop or choose file
3. **Select job profile to run the export** window → select the correct profile for the record type
4. **Are you sure you want to run this job** dialog → confirm **record type** from the dropdown under "UUIDs provided for this export"
5. Click **Run** → job appears in Running section
6. When complete → log row appears in Logs pane
7. Click filename in log → MARC file (.mrc) downloads

### Method 2: Upload CQL query from Data Export app
1. Prepare a .cql file containing a valid CQL query
2. Jobs pane → drag-and-drop or choose file
3. Select job profile
4. **Are you sure you want to run this job** dialog → click **Run**
5. Job completes → log row in Logs pane → click filename to download

### Method 3: Quick export from Inventory
1. Inventory → search/filter for instances
2. Select checkboxes next to desired records
3. Actions → **Export instances (MARC)**
4. UUID .csv file downloads immediately (browser download)
5. MARC .mrc file generated asynchronously → logged in Data Export app → download from there

### Method 4: Quick export from MARC Authority
1. MARC Authority → search for authority records
2. Select checkboxes next to desired records
3. Actions → **Export selected records (CSV/MARC)**
4. UUID .csv file downloads immediately
5. MARC .mrc file logged in Data Export app → download from there

---

## Logs

### Logs pane (main view)
Shows the **25 most recent** export jobs. Each row:

| Column | Description |
|---|---|
| **File name** | Name of the generated MARC file |
| **Status** | Completed / Completed with errors / Fail |
| **Total** | Total records in the export request |
| **Exported** | Number of records successfully exported |
| **Failed** | Number of records that failed |
| **Job profile** | Name of the job profile used |
| **Started running** | Start timestamp |
| **Ended running** | End timestamp |
| **Run by** | Username or system |
| **ID** | System-assigned export job ID |

### Log statuses
| Status | Meaning |
|---|---|
| **Completed** | All records exported successfully |
| **Completed with errors** | Some records exported; some failed; partial MARC file exists |
| **Fail** | No records exported; no MARC file generated |

### View all logs
- Logs pane → **View all** → opens full Logs window
- **Logs are never deleted** — all history retained permanently
- Search by ID; filter by: Errors in export (Yes/No), Started running range, Ended running range, Job profile, User

### Retrieving results
- Completed or Completed with errors → click **File name** to download .mrc
- Fail → no file; click the **row** (not filename) to open error details

### Troubleshooting failed exports
- Clicking the log row opens per-record error log
- Shows: UUID of failed record + error message explaining the failure

---

## On-the-Fly MARC Generation

For instances without an underlying MARC record (source = FOLIO):
- FOLIO generates a simplified MARC record from Inventory metadata
- Less complete than a fully cataloged SRS MARC record
- Holdings and Item data can be appended using transformations in the mapping profile

---

## Transformations in Field Mapping Profiles

Transformations define how Inventory/Holdings/Item fields map to MARC output fields.

### Adding a transformation
1. Field mapping profile → Add transformations
2. Search/filter for the field (e.g. "Holdings - Call number")
3. Select checkbox → fill in: MARC field number, Indicator 1, Indicator 2, Subfield (must start with $)
4. Save & close

### Fields suppression
- Exclude specific MARC tags from the generated output
- Applied after record generation/retrieval

---

## Settings → Data Export

Two settings sections:
- **Job profiles** — create, view, edit, duplicate, delete
- **Field mapping profiles** — create, view, edit, duplicate, delete

### Settings capabilities

| Action | Capability Set |
|---|---|
| View Data Export settings only | `Settings - UI-Data-Export Settings - View` |
| View and edit Data Export settings | `Settings - UI-Data-Export Settings - Edit` |

---

## Required Capability Sets (Eureka — Data Export app)

| Action | Capability Set |
|---|---|
| View export logs only | `Data - UI-Data-Export - View` |
| Upload, export, download, view logs | `Data - UI-Data-Export - Edit` |

**Note:** View allows checking log status only. Edit is required to run exports and download files.

---

## Key Business Rules for Test Cases

1. **Two-level profile hierarchy** — Job profile links to Field Mapping profile; both must exist before running an export
2. **Job profile used in export → cannot be edited or deleted** — must duplicate to create a modified version; Edit and Delete actions are disabled
3. **Field mapping profile assigned to a job profile → cannot be edited or deleted** — must unassign from all linked job profiles (or delete those job profiles) first
4. **Field mapping profile requires at least one FOLIO record type checked** — saving without any type selected returns a validation error
5. **Output format is always MARC** — no alternative output formats exist in the current version
6. **Default authority profile rejects records set for deletion** — each such record produces an individual error in the log; export may still Completed with errors if some records were valid
7. **Deleted authority profile rejects non-deleted records** — same logic in reverse; using the wrong profile always results in per-record errors
8. **Records past the purge window → "Record not found"** — even the correct deleted profile cannot export records purged from SRS; export errors with this message per record
9. **UUID must match the correct record type at confirmation** — staff must select the right record type in the "UUIDs provided for this export" dropdown; wrong type causes systematic export failures
10. **Quick export from Inventory: UUID file immediate, MARC file is async** — .csv of UUIDs arrives in browser immediately; .mrc appears only in Data Export logs
11. **Quick export from MARC Authority: same two-file pattern** — UUID .csv immediate, .mrc async in Data Export logs
12. **Logs are never deleted** — permanent history; 25 most recent shown by default; "View all" needed for older entries
13. **Fail status = no MARC file generated** — clicking filename on a Failed row doesn't download anything; click the row itself to see errors
14. **Completed with errors = partial MARC file exists** — file can be downloaded; failed records documented in error log inside the row
15. **Subfields in transformations must begin with $** — e.g. $a, $b; omitting $ is a validation error when saving the transformation
16. **FOLIO-native instances get simplified generated MARC** — less complete than SRS MARC; important expected-result distinction when comparing MARC source vs FOLIO source instances
17. **CQL file must use .cql extension** — other formats (e.g. .txt) are not accepted as CQL uploads
18. **View capability does not allow running exports or downloading files** — only Edit capability enables the full workflow including upload, run, and download
