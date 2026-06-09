# FOLIO Bulk Edit App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Bulk Edit app before generating test cases.
> Sources: https://docs.folio.org/docs/bulk-edit/

---

## What is Bulk Edit

The Bulk Edit app applies changes to multiple Inventory or User records simultaneously. Maximum **100,000 records** per bulk edit job. It works entirely in-app — no external processing needed.

---

## Record Types Supported

- **Inventory - Holdings**
- **Inventory - Instances**
- **Inventory - Items**
- **Users**

---

## Two Methods to Identify Records

### 1. Identifier (upload a .csv file)
Upload a single-column .csv file of record identifiers. Each record type has its own set of accepted identifiers:

| Record Type | Accepted Identifiers |
|---|---|
| Holdings | Holdings UUIDs, Holdings HRIDs, Instance HRIDs, Item barcodes |
| Instances | Instance UUIDs, Instance HRIDs |
| Items | Item barcodes, Item UUIDs, Item HRIDs, Item former identifiers, Item accession numbers, Holdings UUIDs |
| Users | User UUIDs, User Barcodes, External IDs, Usernames |

**Critical rule:** The .csv must contain **exactly one column**. Multi-column files are rejected with: *"filename is formatted incorrectly. Please correct the formatting and upload the file again."*

### 2. Query (in-app query builder)
Build a query using fields, operators, and values without uploading a file.

**Operators available:**
- equals / not equal to
- contains / contains all / not contains all
- contains any / not contains any
- starts with
- greater than / less than / greater than or equal to / less than or equal to
- is null/empty

**Query must be tested** (Test query button) before it can be run. If results exceed 100,000 records, a warning appears and the query must be narrowed. Building a query requires the `Bulk edit: Can build query` capability.

---

## Bulk Edit Workflow (for both methods)

```
1. Set Criteria (select record type + identifier/query)
2. Preview matching records (Preview of record matched pane)
3. Optional: Download matching records (.csv)
4. Optional: Download errors (.csv)
5. Actions → Start bulk edit
6. Select field(s) to change (Options) + action(s) to apply (Actions)
7. Add more rows with + if editing multiple fields in one job
8. Confirm changes → preview of first 10 records appears ("Are you sure?" modal)
9. Options: Keep editing / Download preview / Commit changes
10. Commit changes → changes applied; success count shown
11. Optional: Download changed records (.csv)
```

---

## Editable Fields by Record Type

### Holdings — editable fields
- **Temporary holdings location** — Replace with (select location) / Clear field
- **Permanent holdings location** — Replace with (select location) / Clear field
- **Suppress from discovery** — Set true / Set false
- **Administrative notes** — Add note / Change note type / Remove all notes / Remove some notes
- **Notes** (holdings notes) — Add note / Change note type / Find & Replace / Remove all / Mark as staff only / Remove mark as staff only

### Instances — editable fields

**Administrative data fields (FOLIO Instances):**
- **Staff suppress** — Set true / Set false
- **Suppress from discovery** — Set true / Set false
- **Cataloged date** — Replace with / Clear field
- **Instance status term** — Replace with
- **Statistical codes** — Add / Remove / Replace

**Instance notes:**
- Add note / Change note type / Find & Replace / Remove all / Mark as staff only / Remove mark as staff only

**MARC Instance fields** (Instances with source = MARC):
- Administrative data (same as above)
- Instance notes
- **0xx–9xx MARC fields** (except control fields 00x and protected 999 ff field)
  - 5XX fields (notes)
  - 9XX fields

### Items — editable fields
- **Temporary item location** — Replace with / Clear field
- **Permanent item location** — Replace with / Clear field
- **Temporary loan type** — Replace with / Clear field
- **Permanent loan type** — Replace with
- **Item status** — Replace with (select status from list)
- **Suppress from discovery** — Set true / Set false
- **Administrative notes** — Add / Change note type / Remove all / Remove some
- **Check in notes** — Add / Change note type / Find & Replace / Remove all / Mark as staff only / Remove mark as staff only
- **Check out notes** — same actions as check in notes
- **Item notes** — Add / Change note type / Find & Replace / Remove all / Mark as staff only / Remove mark as staff only
- **Electronic access** — edit URL relationship, URI, link text, etc.

### Users — editable fields

**In-app (UI) approach:**
- **Expiration date** — Replace with / Clear field
- **Patron group** — Replace with
- **Email** — Replace with

**Local (CSV) approach** (older method, still supported):
- Broader set of user fields editable via CSV upload/download/re-upload cycle
- Requires `Bulk edit: Local - Edit user records` capability (separate from In-app)

---

## Actions Available per Field Type

| Action | Meaning |
|---|---|
| **Replace with** | Replaces current value with new value |
| **Clear field** | Removes the current value (sets to empty) |
| **Set true / Set false** | Toggles boolean fields (suppress, staff suppress) |
| **Add note** | Appends a new note without removing existing ones |
| **Change note type** | Changes the type of existing notes |
| **Find & Replace** | Finds text in a note and replaces it |
| **Remove all** | Deletes all notes of that type |
| **Mark as staff only** | Makes notes visible only to staff |
| **Remove mark as staff only** | Makes notes visible to patrons |

---

## Preview and Confirmation

- After "Confirm changes" a modal shows **preview of first 10 records** to be changed
- Options in the "Are you sure?" modal:
  - **Keep editing** — return to editing without applying
  - **Download preview** — download all changed records as .csv before committing
  - **Commit changes** — apply changes to all matched records
- After Commit: success message shows number of records changed
- Changed records can be downloaded via Actions → Download changed records (CSV)

---

## Errors and Warnings

### Errors during matching (identifier upload)
- Displayed in the **Error accordion** in the Preview pane
- Two columns: Record identifier + Reason for error
- Common cause: identifier in .csv doesn't match any record (record not found, wrong identifier type)
- Can be downloaded as a .csv file

### Warnings during bulk edit
- Displayed as warning messages after Commit
- Indicate records where the change could not be applied
- Warning ≠ failure; other records may still be successfully changed

### 100,000 record limit warning
- If query returns >100,000 records: *"Warning: this query returns more records than the maximum allowed. Modify the query to limit your results."*
- Bulk edit cannot proceed until the query is narrowed

---

## Logs

All bulk edit jobs are logged and accessible via the **Logs** tab in the Bulk Edit app.

### Log columns
- Date / Time
- Record type
- Method (Identifier / Query)
- Status (Completed / Completed with errors / Failed)
- Records processed
- User who ran the job

### Log actions
- **Preview** — view the log details in-app
- **Download** — download the log file (.csv)

Viewing logs requires the `Bulk edit: Can view logs` capability.

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View inventory records (read-only) | `Data - UI-Bulk-Edit Inventory - View` |
| Edit inventory records | `Data - UI-Bulk-Edit Inventory - Edit` |
| View user records via CSV method | `Data - UI-Bulk-Edit Users Csv - View` |
| Edit user records via CSV method | `Data - UI-Bulk-Edit Users Csv - Edit` |
| Edit user records in-app | `Data - UI-Bulk-Edit Users - Edit` |
| View logs | `Data - UI-Bulk-Edit Logs - View` |
| Build query | `Procedural - UI-Bulk-Edit Query - Execute` |
| View base permissions | `Data - UI-Bulk-Edit Base-View - Manage` |
| View edit permissions | `Data - UI-Bulk-Edit Base-Edit - Manage` |

---

## Key Business Rules for Test Cases

1. **Maximum 100,000 records per job** — query or identifier list exceeding this blocks execution with a warning; must narrow scope first
2. **Identifier .csv must have exactly one column** — multi-column files are rejected immediately with a formatting error message before any records are matched
3. **Query must be tested before running** — Test query is a required step; untested queries cannot be committed
4. **Build query requires a separate capability** — `Can build query` is a procedural capability separate from Edit; having Edit alone doesn't unlock the Query tab
5. **In-app and Local (CSV) user editing are separate capabilities** — a user can have one without the other; they use different workflows and editable field sets
6. **Preview shows only first 10 records** — the "Are you sure?" modal previews 10 records; full preview requires downloading before committing
7. **Download preview before commit is optional but recommended** — no automatic preview of all records; user must choose to download
8. **Commit changes is irreversible** — there is no undo after committing; the only recourse is a second bulk edit job to reverse the changes
9. **Multiple fields can be edited in one job** — add rows with the + button to combine multiple field changes in a single commit
10. **Clear field removes value entirely** — different from Replace with empty string; Clear field sets the field to null/empty
11. **Add note doesn't replace existing notes** — appends a new note; to replace, use Find & Replace or Remove all + Add note
12. **Mark as staff only is reversible** — can be undone via "Remove mark as staff only" action on the same note type
13. **MARC 999 ff field is protected** — cannot be edited via Bulk Edit even for MARC instances (same system protection as in Data Import)
14. **Control fields (00x) cannot be edited** — Bulk Edit supports 0xx-9xx MARC fields except control fields
15. **Changed records download is separate from error download** — two distinct CSV files available after job: one for changed records, one for errors
16. **Errors in matching don't block the edit** — if some identifiers don't match records, the error accordion shows them but the bulk edit proceeds on the records that did match
17. **Item status change via Bulk Edit respects Inventory constraints** — not all item statuses are freely settable; some transitions may be restricted by active loans or requests
18. **Logs are permanent** — all bulk edit jobs remain in the Logs tab; useful for audit trails and reviewing past operations
